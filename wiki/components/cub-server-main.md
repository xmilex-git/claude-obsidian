---
type: component
parent_module: "[[modules/src|src]]"
path: "src/executables/server.c"
status: developing
purpose: "cub_server entry point: signal handler registration, argument parsing, and handoff to net_server_start()"
key_files:
  - "server.c (sole file — entire cub_server main lives here)"
public_api:
  - "main(argc, argv) — expects argv[1] = database name"
tags:
  - component
  - cubrid
  - cub-server
  - executables
related:
  - "[[components/executables|executables]]"
  - "[[components/server-boot|server-boot]]"
  - "[[components/connection|connection]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[modules/cubrid|cubrid module]]"
  - "[[sources/cubrid-src-executables|cubrid-src-executables]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cub_server` — Server Entry Point (`server.c`)

`server.c` is the entire `main()` for the `cub_server` process. It is deliberately short: install signal handlers, validate arguments, and call `net_server_start()`. All engine work happens downstream in `boot_sr.c` and the connection layer.

## `main()` Flow

```
main(argc, argv)
  │
  ├─ [Linux/Mac] register_fatal_signal_handler(SIGABRT/SIGILL/SIGFPE/SIGBUS/SIGSEGV/SIGSYS)
  │   crash_handler() → er_print_crash_callstack() + re-raise default
  │   abort_handler() [debug only] → propagate SIGABRT to all clients → abort()
  │
  ├─ [Windows] __try/__except wrapping with CreateMiniDump()
  │
  ├─ argc < 2 → print usage, exit 1
  │
  ├─ envvar_bindir_file() — save full path to this binary (for crash restart)
  │
  ├─ hb_set_exec_path() / hb_set_argv() — HA heartbeat stores exec info for restart
  │   css_set_exec_path() / css_set_argv() — connection layer stores same
  │
  ├─ setsid() — detach from terminal (Linux/Mac only)
  │
  └─ net_server_start(database_name)  ← hands off to engine
       (defined in src/connection/server_support.c / src/communication/)
```

## Signal Semantics

| Signal | Handler | Effect |
|--------|---------|--------|
| `SIGSEGV`, `SIGBUS`, `SIGILL`, `SIGFPE`, `SIGSYS` | `crash_handler` | Print callstack; reset to `SIG_DFL` and re-raise |
| `SIGABRT` (debug builds) | `abort_handler` | Collect pids via `logtb_collect_local_clients()`, send `SIGABRT` to each client, then `abort()` |
| `SIGABRT` (release) | `crash_handler` | Same as other fatal signals |

The HA subsystem (`hb_set_exec_path`, `hb_set_argv`) saves the full command line so `cub_master` can respawn the server process after a crash without needing external orchestration.

## Build Target

`cub_server` is the output of the `cubrid/` top-level CMake target. It compiles with `SERVER_MODE` defined and links the full engine object library. The AGENTS.md binary→source map notes this CMake target as `cubrid/CMakeLists.txt`.

## Key Entry Point Downstream

`net_server_start(database_name)` in `src/connection/server_support.c`:
1. Initialises the thread manager (`cubthread::initialize`)
2. Calls into `boot_sr.c` — the full subsystem init sequence
3. Opens the listening socket pair via `css_tcp_master_open()`
4. Enters the connection dispatch loop

See [[components/server-boot|server-boot]] for the boot sequence and [[components/connection|connection]] for the CSS protocol and socket handling.

## Related

- [[components/executables|executables]] — hub for all binaries
- [[components/server-boot|server-boot]] — the boot sequence `net_server_start()` calls into
- [[Build Modes (SERVER SA CS)]] — `SERVER_MODE` preprocessor guard governs what compiles
- [[modules/cubrid|cubrid module]] — CMake target that produces this binary
