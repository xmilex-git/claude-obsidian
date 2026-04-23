---
type: component
path: "src/win_tools/"
status: developing
purpose: "Windows-only service/tray binaries: NT service host, control CLI, and system-tray GUI for managing CUBRID on Windows"
key_files:
  - "cubridservice/cubridservice.cpp (NT service host: StartServiceCtrlDispatcher, vHandler control dispatcher)"
  - "ctrlservice/ctrlservice.cpp (CLI client: sends control codes to CUBRIDService via SCM)"
  - "cubridtray/CUBRIDtray.cpp (MFC WinApp entry point for CUBRID_Service_Tray.exe)"
  - "cubridtray/NTRAY.CPP (CTrayNotifyIcon wrapper around Shell_NotifyIcon)"
  - "cubridtray/CUBRIDManage.cpp (CCUBRIDManage: process lifecycle for master/server/broker)"
  - "cubridtray/MANAGEREGISTRY.CPP (CManageRegistry: HKEY_LOCAL_MACHINE\\SOFTWARE\\CUBRID reads)"
  - "cubridtray/MAINFRM.CPP (CMainFrame: main window with tray icon and right-click menu)"
tags:
  - component
  - cubrid
  - windows
  - service
  - tray
related:
  - "[[components/porting|porting]]"
  - "[[modules/win|win module]]"
  - "[[components/executables|executables]]"
  - "[[components/broker-impl|broker-impl]]"
created: 2026-04-23
updated: 2026-04-23
---

# `win_tools` — Windows Service & Tray Utilities

`src/win_tools/` is a **Windows-only** subdirectory containing three standalone executables that expose CUBRID process management through the Windows Service Control Manager (SCM) and a system-tray GUI. None of these files are compiled on Linux or macOS — the `win/CMakeLists.txt` that owns them begins with `if(NOT WIN32) return() endif()`.

> [!key-insight] Two-binary control split
> The service implementation is split: `cubridservice.exe` is the actual NT service host registered with the SCM, while `ctrlservice.exe` is a thin CLI client that sends custom control codes to the running service. The `cubrid` admin CLI (on Windows) delegates to `ctrlservice.exe` rather than calling SCM APIs directly.

## Directory Layout

```
src/win_tools/
├── ctrlservice/          → ctrlservice.exe  (CLI control client)
│   ├── ctrlservice.cpp
│   └── stdafx.cpp
├── cubridservice/        → CUBRIDService NT service host
│   ├── cubridservice.cpp
│   └── stdafx.cpp
└── cubridtray/           → CUBRID_Service_Tray.exe (MFC tray app)
    ├── CUBRIDtray.cpp    (CWinApp entry)
    ├── MAINFRM.CPP       (CMainFrame, tray icon, right-click menu)
    ├── NTRAY.CPP         (CTrayNotifyIcon / Shell_NotifyIcon wrapper)
    ├── CUBRIDManage.cpp  (CCUBRIDManage: master/server/broker process control)
    ├── CASManage.cpp     (broker / CAS management)
    ├── MANAGEREGISTRY.CPP (registry helpers)
    ├── Manager.cpp       (CUBRID Manager integration)
    ├── PROCESS.CPP       (CreateProcess wrappers)
    ├── ENV.CPP           (environment variable helpers)
    ├── LANG.CPP          (localization strings)
    └── ... (20+ further MFC dialog/view sources)
```

---

## `cubridservice` — NT Service Host

**Binary name:** `CUBRIDService.exe`
**Service name:** `CUBRIDService` (registered as `SERVICE_WIN32_OWN_PROCESS | SERVICE_INTERACTIVE_PROCESS`, auto-start)

The main entry point calls `StartServiceCtrlDispatcher` and registers `vKingCHStart` as the service main function.

### Startup sequence

1. `SetCUBRIDEnvVar()` — reads `CUBRID`, `CUBRID_DATABASES`, `CUBRID_TMP`, `CUBRID_MODE`, and `Path` from `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment` and injects them via `_putenv`. This is needed because NT services do not inherit the user environment.
2. `vKingCHStart` runs on SCM thread → sets status to `SERVICE_START_PENDING` → fires `cubrid.exe service start --for-windows-service` as a child process (via `CreateProcess`) → sets status to `SERVICE_RUNNING`.
3. The main loop spins on `Sleep(2000)` checking `g_isRunning`.

### Control code handler (`vHandler`)

`vHandler(DWORD opcode)` is the `LPHANDLER_FUNCTION` registered with SCM. It interprets custom control codes (160–223) and delegates to `cubrid.exe <util> <command> --for-windows-service`:

| Code range | Utility | Commands |
|---|---|---|
| 160–163 | `broker` | start / stop / on / off |
| 170–171 | `manager` | start / stop |
| 180–181 | `server` | start / stop (db name via registry key) |
| 190–191 | `service` | start / stop |
| 200–201 | `shard` | start / stop |
| 210–211 | `pl` | start / stop |
| 220–223 | `gateway` | start / stop / on / off |

Database name for per-DB operations is passed through `HKEY_LOCAL_MACHINE\SOFTWARE\CUBRID\CUBRID\CUBRID_DBNAME_FOR_SERVICE` (written by the caller before sending the control code).

After a stop event, `vHandler` calls `SendMessage_Tray` to notify the tray app via `FindWindowA("cubrid_tray", "cubrid_tray")` + `PostMessage`.

---

## `ctrlservice` — CLI Control Client

**Binary name:** `ctrlservice.exe`
**Role:** command-line front-end; it does not run as a service itself.

Usage modes (by `argc`):

| argc | argv[1] | Effect |
|---|---|---|
| 2 | `-i <path>` | Install `CUBRIDService` via `CreateServiceA` with auto-start |
| 2 | `-u` | Uninstall (`DeleteService`) |
| 2 | `-start` | `StartService` on `CUBRIDService` |
| 2 | `-stop` | `ControlService(SERVICE_CONTROL_STOP)` |
| 2 | `-status` | Print `SERVICE_STOPPED` or `SERVICE_RUNNING` |
| 3 | `<util> <cmd>` | Send matching custom control code (160–223) |
| 4 | `server/broker/gateway/pl <cmd> <db>` | Write db name to registry, send control code |

For broker start, the client retries up to 50 × 200 ms if `ERROR_SERVICE_REQUEST_TIMEOUT` is returned — this accommodates slow CAS pool initialization.

> [!note] `ctrlservice` vs `cubridservice` duality
> `ctrlservice/ctrlservice.cpp` and `cubridservice/cubridservice.cpp` share almost identical constant definitions and overlap in service-management logic. `ctrlservice` is the external caller; `cubridservice` is the service that receives the calls.

---

## `cubridtray` — System-Tray GUI

**Binary name:** `CUBRID_Service_Tray.exe` (set via `OUTPUT_NAME` in CMake)
**Framework:** MFC (`CMAKE_MFC_FLAG 2` = shared MFC DLL), requires admin elevation (`requireAdministrator`)
**Window class:** `"cubrid_tray"` (used by `cubridservice` to locate the tray window for `PostMessage` notifications)

### Key classes

| Class | File | Role |
|---|---|---|
| `CUnitrayApp` | `CUBRIDtray.cpp` | `CWinApp` subclass; enforces single-instance via `FindWindow("cubrid_tray", NULL)` |
| `CMainFrame` | `MAINFRM.CPP` | Main (hidden) frame; owns `CTrayNotifyIcon`; handles right-click menu, service start/stop events |
| `CTrayNotifyIcon` | `NTRAY.CPP` | Thin MFC wrapper around `Shell_NotifyIcon` / `NOTIFYICONDATA`; originally by PJ Naughter (1997) |
| `CCUBRIDManage` | `CUBRIDManage.cpp` | Checks if `master.exe` is running; manages DB process list; coordinates start/stop sequencing |
| `CCASManage` | `CASManage.cpp` | Broker / CAS-specific management |
| `CManageRegistry` | `MANAGEREGISTRY.CPP` | Registry helper; reads `HKLM\SOFTWARE\CUBRID\<product>\ROOT_PATH` and related keys |
| `CProcess` | `PROCESS.CPP` | `CreateProcess` wrappers used by manage classes |
| `CLang` | `LANG.CPP` | UI string localization |
| `CEnv` | `ENV.CPP` | Environment variable resolution |

### Tray icon lifecycle

`CMainFrame` uses `WAIT_SERVER(SEC)` macro to animate the tray icon between `IDR_ING` and `IDR_STOP` states while waiting for server start/stop. On receipt of `WM_SERVICE_STOP` / `WM_SERVICE_START` messages from `cubridservice`, it updates the icon accordingly.

### Registry layout (read by tray)

```
HKEY_LOCAL_MACHINE\SOFTWARE\CUBRID\
  CUBRID\
    ROOT_PATH        → CUBRID install directory
    CUBRID_DBNAME_FOR_SERVICE → DB name passed for server start/stop
  CUBRID_JAVA\       (Java Runtime key checked separately)
```

---

## Build Integration

`WIN_TOOLS_DIR` is set to `${CMAKE_SOURCE_DIR}/src/win_tools` in the top-level `CMakeLists.txt`. The three CMake targets (`ctrlservice`, `cubridservice`, `cubridtray`) are defined in `win/CMakeLists.txt` and added only when `WIN32` is true. All three install to `${CUBRID_BINDIR}` (the `bin/` component in the install tree).

The tray target also includes `version.h` from the `win/` directory (noted as a TODO to remove).

---

## Relationship to Other Components

- **[[components/porting|porting]]** — Win32 shims (`EXPORT_IMPORT`, `poll()`, POSIX aliases) used by the engine but not directly by win_tools, which calls Win32 APIs natively.
- **[[modules/win|win module]]** — `win/CMakeLists.txt` is the build owner; `win/` also ships 3rd-party DLLs (jansson, libexpat, lz4, re2) installed alongside these binaries.
- **[[components/executables|executables]]** — `cubrid.exe` (from `src/executables/`) is what `cubridservice` and `ctrlservice` shell out to via `CreateProcess`; the `--for-windows-service` flag signals it to behave headlessly.
- **[[components/broker-impl|broker-impl]]** — Broker start/stop is a primary use case; control codes 160–163 correspond exactly to broker lifecycle operations.

## Related

- Source: [[sources/cubrid-src-win-tools|cubrid-src-win-tools]]
