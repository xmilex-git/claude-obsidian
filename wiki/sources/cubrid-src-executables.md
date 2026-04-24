---
created: 2026-04-23
type: source
title: "CUBRID src/executables/ — Binary Entry Points"
source_path: "src/executables/"
ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - executables
  - binaries
related:
  - "[[components/executables|executables]]"
  - "[[components/cub-server-main|cub-server-main]]"
  - "[[components/csql-shell|csql-shell]]"
  - "[[components/cub-master-main|cub-master-main]]"
  - "[[components/utility-binaries|utility-binaries]]"
  - "[[components/cub-master|cub-master]]"
  - "[[Build Modes (SERVER SA CS)]]"
---

# Source: `src/executables/`

Binary entry points for all CUBRID processes. Files read and summarized: `AGENTS.md`, `server.c`, `csql.c`, `csql_launcher.c`, `master.c`, `util_service.c`, `util_cs.c`, `util_sa.c`, `util_common.c`, `unloaddb.c`, `compactdb.c`.

## Key Files Read

| File | Binary | Lines skimmed |
|------|--------|--------------|
| `server.c` | `cub_server` | full `main()` |
| `csql_launcher.c` | `csql` | full `main()`, DSO load |
| `csql.c` | `csql` REPL | globals, session cmd decls |
| `master.c` | `cub_master` | full `main()`, `css_master_loop()`, `css_process_new_connection()` |
| `util_service.c` | `cubrid` | enum + dispatch structure |
| `util_cs.c` / `util_sa.c` | admin utils | includes + function signatures |
| `util_common.c` | shared | `utility_initialize()` |
| `unloaddb.c` | `unloaddb` | top of file |
| `compactdb.c` | `compactdb` | top of file |

## Pages Created

- [[components/executables|executables]] — component hub
- [[components/cub-server-main|cub-server-main]] — `server.c` entry point
- [[components/csql-shell|csql-shell]] — CSQL REPL design
- [[components/cub-master-main|cub-master-main]] — `master.c` main loop
- [[components/utility-binaries|utility-binaries]] — admin utility overview

## Key Insights

1. **DSO pattern in csql**: `csql_launcher.c` `dlopen()`s either `libutil_sa` or `libutil_cs` at runtime and calls `csql()` by symbol name — enabling a single binary to operate in SA or CS mode.
2. **cub_master is single-threaded**: The entire `css_master_loop()` runs in a single thread on a `select()` loop with 4-second timeout; no worker pool.
3. **`util_cs.c` includes both `boot_cl.h` and `boot_sr.h`**: Some admin utilities (restoredb) use SA_MODE paths inside an otherwise CS-mode compilation unit.
4. **Auto-restart**: `master_server_monitor.hpp` / `server_monitor` allows `cub_master` to automatically respawn a crashed `cub_server` — gated by `auto_restart_server` cubrid.conf parameter.
