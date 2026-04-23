---
type: source
title: "CUBRID src/win_tools/ — Windows Service & Tray Tools"
source_path: "src/win_tools/"
date_ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - windows
  - service
  - tray
related:
  - "[[components/win-tools|win-tools]]"
  - "[[components/porting|porting]]"
  - "[[modules/win|win module]]"
---

# Source: `src/win_tools/`

**Ingested:** 2026-04-23
**Coverage:** All three subdirectories — `ctrlservice/`, `cubridservice/`, `cubridtray/` — plus build integration via `win/CMakeLists.txt` and the top-level `CMakeLists.txt` (`WIN_TOOLS_DIR` variable).

## What Was Read

| File | Notes |
|---|---|
| `win/CMakeLists.txt` | Build definitions for all three targets; `WIN_TOOLS_DIR`, install rules, 3rd-party DLL installs |
| `CMakeLists.txt` (root) | `WIN_TOOLS_DIR` assignment, `add_subdirectory(win)` guard |
| `cubridservice/cubridservice.cpp` | NT service host: `main`, `vKingCHStart`, `vHandler`, `SetCUBRIDEnvVar`, `SendMessage_Tray`, `proc_execute` |
| `ctrlservice/ctrlservice.cpp` | CLI client: `_tmain`, `vctrlService`, `vDelService`, `vStartService`, `vStopService`, `vPrintServiceStatus`, `write_string_value_in_registry` |
| `cubridtray/CUBRIDtray.cpp` | `CUnitrayApp` MFC app entry, single-instance guard |
| `cubridtray/MAINFRM.CPP` | `CMainFrame`, tray icon animation, WM_SERVICE_* message handlers |
| `cubridtray/NTRAY.CPP` | `CTrayNotifyIcon` (`Shell_NotifyIcon` wrapper) |
| `cubridtray/CUBRIDManage.cpp` | `CCUBRIDManage` process lifecycle management |
| `cubridtray/MANAGEREGISTRY.CPP` | `CManageRegistry` registry read helpers |

## Key Findings

1. **Three binaries, two roles.** `cubridservice.exe` is the NT service host. `ctrlservice.exe` is the CLI client that sends SCM custom control codes to it. `CUBRID_Service_Tray.exe` is an independent MFC GUI that listens for `WM_SERVICE_*` window messages from the service host.

2. **Custom control codes (160–223) bridge the CLI ↔ service gap.** Rather than spawning separate processes per subsystem, the design routes all lifecycle events (broker, server, gateway, shard, manager, pl) through a single `CUBRIDService` SCM handle using out-of-range control codes.

3. **Environment injection at service startup.** Because NT services run in a bare environment, `SetCUBRIDEnvVar()` manually reads `CUBRID`, `CUBRID_DATABASES`, `CUBRID_TMP`, `CUBRID_MODE`, and `Path` from the system registry and injects them via `_putenv` before any child process is spawned.

4. **Registry as inter-process parameter passing.** The database name for per-DB server start/stop is written to `HKLM\SOFTWARE\CUBRID\CUBRID\CUBRID_DBNAME_FOR_SERVICE` by `ctrlservice` immediately before sending the control code, and read back by `cubridservice`'s handler. This is a deliberate (if fragile) IPC mechanism.

5. **MFC tray app is a 1997-era design.** `NTRAY.CPP` credits PJ Naughter (1997) and uses `Shell_NotifyIcon` / `NOTIFYICONDATA` directly. The tray app communicates with the service only via `FindWindowA` + `PostMessage` — no shared memory or named pipes.

6. **`--for-windows-service` flag.** The `cubrid` admin binary (`cubrid.exe` from `src/executables/`) accepts this flag to suppress interactive output and behave headlessly when called as a child process from the NT service.

## Pages Created

- [[components/win-tools|win-tools]] — component hub page
