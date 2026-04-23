---
type: component
parent_module: "[[modules/src|src]]"
path: "src/executables/util_*.c, unloaddb.c, compactdb.c, commdb.c"
status: developing
purpose: "Admin utility binaries shipped with CUBRID: database lifecycle management (create, delete, copy, restore, backup), space inspection, lock display, bulk export, space compaction, HA utilities"
key_files:
  - "util_service.c (cubrid service front-end: start/stop/status for all subsystems)"
  - "util_cs.c (CS_MODE helpers: backupdb, restoredb, spacedb, killtran, lockdb, diagdb, …)"
  - "util_sa.c (SA_MODE helpers: createdb, deletedb, copydb, optimizedb, plandump, …)"
  - "util_common.c (arg parsing, DB name validation, HA parameter helpers)"
  - "unloaddb.c (unloaddb: export schema + data in loaddb format)"
  - "compactdb.c (compactdb: reclaim OID holes in heap files)"
  - "commdb.c (commdb: query cub_master for registered servers and status)"
public_api:
  - "utility_initialize() — er_init + msgcat_init; called by all utilities"
tags:
  - component
  - cubrid
  - utilities
  - admin
  - executables
related:
  - "[[components/executables|executables]]"
  - "[[components/server-boot|server-boot]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[modules/sa|sa module]]"
  - "[[modules/cs|cs module]]"
  - "[[sources/cubrid-src-executables|cubrid-src-executables]]"
created: 2026-04-23
updated: 2026-04-23
---

# Admin Utility Binaries

CUBRID ships a set of admin utilities for database lifecycle, space management, lock inspection, and data migration. All utilities share a common pattern: parse arguments → call `utility_initialize()` → dispatch to `util_cs.c` or `util_sa.c` → exit.

## `cubrid` — Service Front-End (`util_service.c`)

`util_service.c` is the universal entry point for all service-level operations. It dispatches based on a first-level subcommand:

```
cubrid service  {start|stop|status|restart|...}
cubrid server   {start|stop|status} <dbname>
cubrid broker   {start|stop|status|...}
cubrid manager  {start|stop|status}
cubrid heartbeat {start|stop|status|...}
cubrid admin    <util-subcommand> ...
cubrid pl       {start|stop|status}    (Java PL engine)
cubrid gateway  {start|stop|status}
```

Internally, `util_service.c` defines `UTIL_SERVICE_INDEX_E` (SERVICE=0, SERVER=1, BROKER=2, MANAGER=3, HEARTBEAT=4, ADMIN=8, PL_UTIL=20, GATEWAY=21) and dispatches. For server start/stop it forks and execs `cub_server` / `cub_master` directly. Admin commands (`cubrid admin`) forward to `util_cs.c` or `util_sa.c` functions.

## SA-Mode Utilities (`util_sa.c`)

These mount the database in-process (SA_MODE, link `cubridsa`) — no running server required:

| Utility | Function | SA_MODE key API |
|---------|---------|-----------------|
| `createdb` | Create new database with initial volumes | `boot_restart_server()` + DDL |
| `deletedb` | Delete all database volumes and logs | `boot_delete()` |
| `copydb` | Copy/rename a database | `boot_copy()` |
| `optimizedb` | Update statistics for the optimizer | SA bootstrap + catalog update |
| `installdb` | Register a database in `databases.txt` | file operations only |
| `plandump` | Dump cached query execution plans | SA bootstrap + plan cache access |
| `compactdb` | Reclaim OID holes in heap files | see `compactdb.c` below |
| `migdb` | Migrate database to new CUBRID version | SA bootstrap + schema transform |
| `localedb` | Compile locale data and generate libraries | locale/timezone support |
| `timezonedb` | Compile timezone data | `tz_compile.c` |
| `tranlist` | List active transactions | SA + transaction table |

`util_sa.c` includes `boot_sr.h`, `heap_file.h`, `btree.h`, `locator_sr.h`, `xserver_interface.h` — it directly accesses server-side engine APIs, which is valid in SA_MODE (client and server in the same process).

## CS-Mode Utilities (`util_cs.c`)

These connect to a running `cub_server` via the CSS protocol (CS_MODE, link `cubridcs`):

| Utility | Function | CS key API |
|---------|---------|-----------|
| `backupdb` | Online backup | `net_client_request_with_callback()` to server |
| `restoredb` | Restore from backup | SA_MODE path in util_sa.c; CS queries master |
| `spacedb` | Show volume/space usage | server RPC |
| `killtran` | Kill a transaction by index | `logtb_kill_or_interrupt_tran()` via RPC |
| `lockdb` | Display lock table | `lock_dump()` via RPC |
| `diagdb` | Diagnostic dump (heap, btree, catalog) | RPC |
| `checkdb` | Verify database integrity | `boot_check_db_consistency()` via RPC |
| `flashback` | Flashback / time-travel query | `flashback_cl.c` |
| `dumpdb` | Dump raw log records | log applier path |
| `applyinfo` | Show HA log apply status | HA CS utilities |

`util_cs.c` includes both `boot_cl.h` (client boot) and `boot_sr.h` (server boot) because some paths run SA_MODE (e.g. restore), while others use CSS connections. The `connection_support.hpp` include provides `css_readn`/`css_writen` for sending admin packets.

## `unloaddb.c`

Entry point for `unloaddb`, which exports schema DDL and data in `loaddb` format.

- Links `cubridsa` (SA_MODE).
- Calls `extract_schema.hpp` for schema DDL output.
- Optionally uses `fork()`-based multi-processing (`MULTI_PROCESSING_UNLOADDB_WITH_FORK`) for parallel object export (up to `MAX_PROCESS_COUNT = 36` workers).
- Output: `<db>_schema` (DDL) and `<db>_objects` (data) files.

## `compactdb.c`

Entry point for `compactdb`, which reclaims OID holes left by MVCC deletes.

- Links `cubridsa` (SA_MODE).
- Calls `compactdb_start(verbose, input_file, class_list, class_count)`.
- Internally uses `heap_file.c` and `btree.c` APIs directly via SA_MODE access.
- Tracks `class_objects`, `total_objects`, `failed_objects` counters.
- Accepts an optional class list (`-i file` or `-c class ...`) to compact only specified tables.

## `commdb.c`

`commdb` is a light management client that connects to `cub_master` as an `INFO_REQUEST` client and queries status. Used internally by `cubrid server status` and similar commands. Linked against `cubridcs` (CS_MODE); talks the CSS info protocol.

## `util_common.c` — Shared Helpers

Provides:
- `utility_initialize()` — called by every utility at startup: `er_init` + `msgcat_init`
- `utility_get_option_index()` — option argument map lookup
- `check_database_name_local()` — validates DB name against `databases.txt` (exists or new)
- `util_split_ha_node/db/sync()` — parse HA config strings
- `util_get_ha_parameters()` — aggregate HA config from `cubrid.conf`
- `util_is_replica_node()` — detect replica role for HA

## Common Pattern

Every utility's `main()` follows the same skeleton:

```c
int main(int argc, char **argv) {
    if (utility_initialize() != NO_ERROR) return EXIT_FAILURE;
    // parse argv with GETOPT_LONG / UTIL_ARG_MAP
    // call into util_sa.c or util_cs.c
    // msgcat_final(); er_final(ER_ALL_FINAL);
    return EXIT_SUCCESS;
}
```

Utilities that need SA_MODE call `boot_restart_server()` / `db_restart()` with `SA_MODE` defined at compile time. Utilities that need CS_MODE call `db_restart()` which connects via CSS to the running server.

## Related

- [[components/executables|executables]] — hub for all binaries
- [[Build Modes (SERVER SA CS)]] — SA vs CS link-time selection
- [[modules/sa|sa module]] / [[modules/cs|cs module]] — CMake targets producing `libutil_sa` / `libutil_cs`
- [[components/loaddb|loaddb]] — the loaddb utility (opposite of unloaddb) lives in `src/loaddb/`
- [[components/server-boot|server-boot]] — `boot_sr.c` APIs used by SA-mode utilities
