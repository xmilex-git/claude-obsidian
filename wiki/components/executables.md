---
type: component
parent_module: "[[modules/src|src]]"
path: "src/executables/"
status: developing
purpose: "All CUBRID binary entry points: cub_server, csql, cub_master, admin utilities (createdb, unloaddb, compactdb, …), and the cubrid service front-end"
key_files:
  - "server.c (cub_server main)"
  - "csql_launcher.c (csql entry point — arg parse + DSO load)"
  - "csql.c (csql REPL engine)"
  - "master.c (cub_master main + select loop)"
  - "master_heartbeat.c (HA heartbeat inside master process)"
  - "util_service.c (cubrid service subcommand dispatcher)"
  - "util_cs.c (CS_MODE utility helpers: backupdb, spacedb, killtran, …)"
  - "util_sa.c (SA_MODE utility helpers: createdb, deletedb, copydb, restoredb, optimizedb, …)"
  - "util_common.c (arg parsing, DB name validation, HA node helpers)"
  - "unloaddb.c (unloaddb entry point)"
  - "compactdb.c (compactdb entry point)"
  - "commdb.c (commdb: master status query utility)"
public_api:
  - "main() in server.c → net_server_start(db_name)"
  - "main() in csql_launcher.c → csql() via DSO"
  - "main() in master.c → css_master_loop()"
  - "utility_initialize() in util_common.c"
tags:
  - component
  - cubrid
  - executables
  - binaries
related:
  - "[[modules/src|src]]"
  - "[[components/cub-server-main|cub-server-main]]"
  - "[[components/csql-shell|csql-shell]]"
  - "[[components/cub-master-main|cub-master-main]]"
  - "[[components/utility-binaries|utility-binaries]]"
  - "[[components/cub-master|cub-master]]"
  - "[[components/server-boot|server-boot]]"
  - "[[components/cas|cas]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[sources/cubrid-src-executables|cubrid-src-executables]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/executables/` — Binary Entry Points & Utilities

Every CUBRID binary program starts in this directory. The files here are thin entry points: they parse arguments, set up signal handlers, then hand off to engine libraries (`cubrid/`, `cs/`, `sa/` CMake targets).

## Binary Inventory

| Binary | Entry file | Build target / link library |
|--------|-----------|------------------------------|
| `cub_server` | `server.c` | `cubrid/` CMake target |
| `csql` | `csql_launcher.c` → `csql.c` | `cs/` (CS_MODE) or `sa/` (SA_MODE) via DSO |
| `cub_master` | `master.c` | top-level CMake |
| `cubrid` | `util_service.c` | top-level CMake |
| `loaddb` | `src/loaddb/` (own grammar) | `sa/CMakeLists.txt` |
| `unloaddb` | `unloaddb.c` | `sa/CMakeLists.txt` |
| `compactdb` | `compactdb.c` | `sa/CMakeLists.txt` |
| `commdb` | `commdb.c` | top-level CMake |

## Architecture

```
 User shell
     │
     ├── csql (SA: links cubridsa; CS: links cubridcs via DSO)
     │     └── csql_launcher.c → utility_load_library(LIB_UTIL_SA_NAME / LIB_UTIL_CS_NAME)
     │           └── csql() in csql.c (REPL loop)
     │
     ├── cub_server db_name
     │     └── server.c:main() → net_server_start(db_name)  [SERVER_MODE]
     │           └── boot_sr.c (server-boot sequence)
     │
     ├── cub_master [port]
     │     └── master.c:main() → css_master_loop()  [select() event loop]
     │
     ├── cubrid [service|server|broker|…] [start|stop|status|…]
     │     └── util_service.c → forks child binaries / calls util_cs.c / util_sa.c
     │
     └── unloaddb / compactdb / …
           └── *.c:main() → util_sa.c or util_cs.c helpers → cubridsa / cubridcs
```

## Link-Time Build Mode Split

| Binary | Mode | Library | What it can do |
|--------|------|---------|----------------|
| `cub_server` | `SERVER_MODE` | links against engine objects directly | I/O, thread pools, full SQL engine |
| `cub_master` | none (standalone) | `connection_globals`, `tcp`, `client_support` | socket management only |
| `csql -S` (SA) | `SA_MODE` | `cubridsa` | in-process SQL, no remote server needed |
| `csql -C` (CS) | `CS_MODE` | `cubridcs` | connects to running `cub_server` via CSS |
| `loaddb/unloaddb/compactdb` | `SA_MODE` | `cubridsa` | direct heap/btree access |
| `util_cs.c` utilities | `CS_MODE` | `cubridcs` | remote admin commands (backupdb, killtran, …) |

See [[Build Modes (SERVER SA CS)]] for the preprocessor guard mechanism.

## Signal Handling in `server.c`

`cub_server` installs fatal signal handlers before calling `net_server_start()`:

- `SIGSEGV`, `SIGBUS`, `SIGILL`, `SIGFPE`, `SIGSYS` → `crash_handler()` — prints callstack via `er_print_crash_callstack()`
- `SIGABRT` → `abort_handler()` (debug builds) — propagates SIGABRT to all connected clients via `logtb_collect_local_clients()`, then aborts
- Windows: `CreateMiniDump()` via SEH `__try`/`__except`

## Sub-Component Pages

- [[components/cub-server-main|cub-server-main]] — `server.c` entry point, signal setup, `net_server_start()` call
- [[components/csql-shell|csql-shell]] — CSQL interactive REPL design, session commands, DSO loading
- [[components/cub-master-main|cub-master-main]] — `master.c` main loop (`css_master_loop`), connection dispatch
- [[components/utility-binaries|utility-binaries]] — admin utilities: createdb, unloaddb, compactdb, cubrid service front-end

## Related

- [[components/cub-master|cub-master]] — protocol-side master documentation (from `src/connection/`)
- [[components/server-boot|server-boot]] — `boot_sr.c` subsystem init, called by `cub_server` after `main()`
- [[components/cas|cas]] — broker CAS worker processes (separate binary in `src/broker/`)
- Source: [[sources/cubrid-src-executables|cubrid-src-executables]]
