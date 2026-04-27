---
created: 2026-04-27
type: source
title: "CUBRID Manual — Administration Guide (admin/)"
source_path: "/home/cubrid/cubrid-manual/en/admin/ — control.rst, admin_utils.rst, db_manage.rst, scripts.rst, troubleshoot.rst, ddl_audit.rst, systemtap.rst (excludes config.rst and error_log_*.rst — see separate pages)"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - admin
  - operations
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-config-params]]"
  - "[[sources/cubrid-manual-error-codes]]"
  - "[[sources/cubrid-manual-ha]]"
  - "[[sources/cubrid-manual-security]]"
  - "[[components/executables]]"
  - "[[components/cub-master]]"
  - "[[components/heartbeat]]"
  - "[[components/recovery]]"
---

# CUBRID Manual — Administration Guide

**Ingested:** 2026-04-27
**Source files:** `admin/index.rst` (136), `control.rst` (3395), `admin_utils.rst` (3898 + included `backup.inc` ~1057, `migration.inc` ~1000), `db_manage.rst` (159), `scripts.rst` (263), `troubleshoot.rst` (286), `ddl_audit.rst` (78), `systemtap.rst` (452)
**Companion pages:** [[sources/cubrid-manual-config-params]] for `config.rst`; [[sources/cubrid-manual-error-codes]] for `error_log_*.rst`.

## Section Map

| Section | File | Content |
|---|---|---|
| Index | `index.rst` | Toctree: service-mgmt utilities + DB-mgmt utilities |
| Operational manual | `control.rst` | `cubrid service/server/broker/gateway/manager/heartbeat/pl` verbs; broker_status, gateway_status, broker_log_top, cubrid_replay; broker ACL; SSL; SHARD broker; **server event log** (SLOW_QUERY/MANY_IOREADS/LOCK_TIMEOUT/DEADLOCK/TEMP_VOLUME_EXPAND); CAS error catalogue (-10001..-10200); PL server lifecycle; `cubrid_utility.log` |
| DB management overview | `db_manage.rst` | `databases.txt` format + `CUBRID_DATABASES` env; file taxonomy: data, temp (`<db>_t32766..`), active log (`<db>_lgat`), archive logs (`<db>_lgar###`), bg-archive log (`<db>_lgar_t`), DWB file (`<db>_dwb`), TDE keys file (`<db>_keys`), volume info (`<db>_vinf`), log info (`<db>_lginf`) |
| Utility reference | `admin_utils.rst` | Full reference for every `cubrid <util>`: createdb, addvoldb, deletedb, renamedb, alterdbhost, copydb, installdb, spacedb, compactdb, optimizedb, plandump, statdump, lockdb, tranlist, killtran, checkdb, diagdb, paramdump, **tde**, vacuumdb, **flashback**, **memmon** (new in 11.4); HA/locale/timezone command summaries; backup.inc (backupdb/restoredb), migration.inc (unloaddb/loaddb) |
| Backup utilities | `backup.inc` | backupdb (online/offline, levels 0/1/2, --compress, --sleep-msecs, -k --separate-keys); restoredb (-d up-to-date, --list, -B, -p partial, -u, -k); cross-server restore |
| Migration utilities | `migration.inc` | unloaddb, loaddb (grammar, command-line + data-file syntax, schema/index/trigger separation) |
| Helper scripts | `scripts.rst` | `unloaddb.sh` (DBA-only, parallel unload, default 8 / max 16 procs, `-i/-D/-s/-d/-v`); `restore_to_newdb.sh` (Linux-only, restores backup as new DB name, registers in databases.txt). Both at `$CUBRID/share/scripts` |
| Troubleshoot | `troubleshoot.rst` | Find SQL log when CAS error occurs (CAS_INFO from `cci_get_cas_info`/JDBC `toString`); slow-query analysis; server error log location (`$CUBRID/log/server/<db>_<yyyymmdd>_<hhmi>.err`); overflow detection (`-1125/-1126/-1127`); log-recovery timing (`-1128/-1129`); deadlock messages; HA split-brain detection; copylogdb/applylogdb unrestorable cases |
| DDL audit | `ddl_audit.rst` | `ddl_audit_log=on`, `ddl_audit_log_size`; logs at `$CUBRID/log/ddl_audit/`; per-source file naming + pipe-separated record format |
| SystemTap | `systemtap.rst` | Install (≥2.2, stapusr/stapdev), build flag `ENABLE_SYSTEMTAP=ON` (default), markers: connection (conn_start/end), query (query_exec_start/end), object/index ops, lock (lock_acquire/release_start/end), txn (tran_commit/abort/start/deadlock), I/O (pgbuf_hit/miss, io_read/write_start/end), sort (sort_start/end); sample at `$CUBRID/share/systemtap/tapset/scripts/buffer_access.stp` |

## Utility Inventory (`cubrid <verb>`)

### Service / process control
- `cubrid service` — start/stop/status the configured `service` set (master, broker, manager, [server], [pl])
- `cubrid server` — start/stop/status/restart specific DB. **In HA mode use `cubrid heartbeat` instead** to avoid bypassing failover orchestration
- `cubrid broker` — start/stop/status/reset/acl, broker_status (real-time stats)
- `cubrid gateway` — DBLink heterogeneous gateway control (start/stop/status/acl/on/off/reset/info)
- `cubrid manager` — CUBRID Manager server start/stop/status
- `cubrid heartbeat` — HA: start/stop/status/reload (also `copylogdb`/`applylogdb` subverbs to control individually)
- `cubrid pl` — PL server (`cub_pl`) start/stop/status (managed by cub_server by default; explicit control rarely needed)
- `cubrid changemode` — view/change server status (active/standby/maintenance)

### Database management
- `cubrid createdb <db> <locale>` — create new DB. Default 1.5 GB volumes (data 512 + active log 512 + bg-archive 512). Locale is permanent.
- `cubrid addvoldb -p data|temp <db>` — add data or temp volume. Permanent volumes assigned for temp data take priority over real temp volumes.
- `cubrid deletedb` — drop DB
- `cubrid renamedb` / `cubrid alterdbhost` — DB metadata changes
- `cubrid copydb` — clone DB
- `cubrid installdb` — register an existing DB at a new location into `databases.txt`
- `cubrid spacedb` — disk usage report (improved in 11.4: classifies overflow pages as INDEX/HEAP rather than SYSTEM; ADMIN_LOG_FILE display added; absolute paths shown)
- `cubrid compactdb` — heap reclamation
- `cubrid optimizedb` — runs UPDATE STATISTICS (sampling 5000 pages by default in 11.4, was 1000 before)
- `cubrid plandump` — dump cached query plans (`-d` delete; `-s <plan_id>` delete specific in 11.4)
- `cubrid statdump -i N -c` — periodic performance counter dump (per-second, cumulative)
- `cubrid paramdump` — current effective system parameter values

### Transaction / lock control
- `cubrid lockdb <db>` — list active locks
- `cubrid tranlist <db>` — list transactions
- `cubrid killtran -i <id>` — terminate a transaction

### Diagnostics
- `cubrid checkdb` — consistency check
- `cubrid diagdb -d <type> <db>` — internal dump. Types: -1 all, 1 file table, 2 file capacity, 3 heap capacity, 4 index capacity, 5 class names, 6 disk bitmap, 7 catalog, 8 log, **9 (new in 11.4) heap data for a specific class** (use with `-n <class>`)
- `cubrid memmon <db>` (NEW in 11.4) — heap memory profiling. Requires `enable_memory_monitoring=yes` + server restart. Per-source-line allocation tracking
- `cubrid flashback <db>` (DBA-only) — point-in-time data recovery from supplemental log. Requires `supplemental_log` ON. 300 s `flashback_timeout` (mandatory). One-at-a-time. CS mode only. SETs/MULTISETs/LISTs/JSON/LOB report NULL in output (limitation)

### Backup / restore (backup.inc)
- `cubrid backupdb -S <db>` — online backup. `-l N` for incremental (level 1/2). `--no-compress` to disable compression (default ON since 11.2). `--sleep-msecs=N` throttles live backups (sleep N ms per 1 MB read). `-t COUNT` thread count (default = #CPUs). `-r` removes pre-backup archive logs. `-k --separate-keys` exports TDE key file separately
- `cubrid restoredb` — restore. `-d backuptime` token = restore to last backup. `-d 'dd-mm-yyyy:hh:mi:ss'` for point-in-time. `--list` shows backup metadata without restoring. `-l N` level for incremental. `-k --keys-file-path` for separate TDE key file

### Migration (migration.inc)
- `cubrid unloaddb -S -u dba <db>` — exports schema + objects + indexes to files
- `cubrid loaddb -u dba -s <schema> -d <objects> -i <indexes> <db>` — imports back

### TDE (`cubrid tde`)
- `--show-keys` — list keys, current active, timestamps
- `--add-key` — generate new master key
- `--delete-key <idx>` — delete (cannot delete active key)
- `--change-key <idx>` — switch active master key
- Key file: `<db>_keys` (default at data volume location, override via `tde_keys_file_path`)

### Other
- `cubrid vacuumdb -C --dump <db>` — VACUUM diagnostic; reports first log page still referenced + OVFP-threshold violators
- `cubrid genlocale` / `dumplocale` — locale build / inspect (uses `make_locale.sh`)
- `cubrid applyinfo` — HA replication progress

## Process Files & Naming

| File | Path | Owner |
|---|---|---|
| Server error log | `$CUBRID/log/server/<db>_<yyyymmdd>_<hhmi>.err` | cub_server |
| Server event log | `$CUBRID/log/server/<db>_<yyyymmdd>_<hhmi>.event` | cub_server |
| PL server log | `$CUBRID/log/<db>_pl.{err,log}` | cub_pl |
| `cubrid_utility.log` | `$CUBRID/log/` | All `cubrid <util>` invocations; rotates at `error_log_size`; `.bak` backup |
| DDL audit log | `$CUBRID/log/ddl_audit/` | CAS / csql / loaddb (per-source file name) |
| Master log | `$CUBRID/log/<host>.cub_master.err` | cub_master |
| Broker SQL log | `$CUBRID/log/broker/sql_log/` | CAS, rotates at `SQL_LOG_MAX_SIZE` |
| `databases.txt` | `$CUBRID_DATABASES/databases.txt` | All utilities | 

## Notable Facts

- **Auto-restart in non-HA**: 11.4 NEW — `auto_restart_server=yes` enables cub_master to auto-restart cub_server after abnormal termination (OOM kill, segfault). Disabled if too-frequent crashes (within 120 s) or repeated start failures.
- **PL server (cub_pl) auto-managed**: starts/stops with cub_server when `stored_procedure=yes` (default). Manual `cubrid pl start/stop` is rare.
- **`cubrid server start` in HA mode = trap**: bypasses heartbeat. Operators in HA must use `cubrid heartbeat start`.
- **Backup is compressed by default since 11.2** (`--no-compress` to disable).
- **Server event log** (`<db>_<...>.event`) records 5 event types: SLOW_QUERY, MANY_IOREADS, LOCK_TIMEOUT, DEADLOCK, TEMP_VOLUME_EXPAND. Triggers controlled by `sql_trace_slow`, `sql_trace_ioread_pages`, `lock_timeout`.
- **Deadlock event in 11.4** records ONLY the lock info contributing to the deadlock (simplified output).
- **PL server `-900/-901` errors** — handshake failures with cub_server.
- **CAS error code range `-10001..-10200`** — distinct from server errors. `-10026 CAS_ER_MAX_PREPARED_STMT_COUNT_EXCEEDED` when broker `MAX_PREPARED_STMT_COUNT` exceeded.
- **`unloaddb.sh`** = parallel wrapper around `cubrid unloaddb`. 8 default child procs, max 16. CTRL-C deletes in-progress object files but preserves completed ones.
- **`restore_to_newdb.sh`** (Linux only) — restores a backup as a new DB name and rewrites `databases.txt` (backing up to `databases.txt.bak`). Produces log `restoredb_YYYYMMDDhhmiss.log`. Requires `-k <keys-file>` for TDE backups.
- **SystemTap markers** are listed by category. Build flag `ENABLE_SYSTEMTAP=ON` is the **default** in shipped CUBRID packages.

## Cross-References

- [[sources/cubrid-manual-config-params]] — every `cubrid.conf`/`cubrid_broker.conf` parameter
- [[sources/cubrid-manual-error-codes]] — full error catalogue (792 codes)
- [[sources/cubrid-manual-ha]] — `cubrid heartbeat`, `cubrid changemode`, `cubrid applyinfo`
- [[sources/cubrid-manual-security]] — `cubrid tde`, ACL, SSL
- [[components/executables]] — implementation of `cubrid <verb>` dispatcher
- [[components/cub-master]] · [[components/cub-master-main]] — auto-restart logic
- [[components/recovery]] — `-1128/-1129` log recovery NOTIFICATIONs
- [[components/heartbeat]] — HA-related events

## Incidental Wiki Enhancements

- [[components/cub-master]]: documented `auto_restart_server` (NEW in 11.4) — cub_master auto-restarts cub_server after abnormal termination, with 120 s repeat-crash + retry-count guards.
- [[components/recovery]]: documented log-recovery NOTIFICATION codes -1128 (start, with "log records to be applied: N, log page: A~B") / -1129 (finish), and `recovery_progress_logging_interval` cadence.
- [[components/lock-manager]]: documented `lock_escalation` default 100,000 + `rollback_on_lock_escalation` flag; `lock_timeout=-1` infinite, `0` no-wait; LOCK_TIMEOUT event log records waiter+blocker SQL+bind values.
- [[components/deadlock-detection]]: documented `deadlock_detection_interval_in_secs` default 1.0 s (min 0.1, rounded up to 0.1); SystemTap `tran_deadlock()` probe; 11.4 simplified `<db>_latest.event` deadlock output (only contributing locks).
- [[components/vacuum]]: documented `cubrid vacuumdb -C --dump` output: first log page still referenced + OVFP read-threshold violators per index; tied to `vacuum_ovfp_check_threshold` (1000) and `vacuum_ovfp_check_duration` (45000); `vacuum_prefetch_log_buffer_size` 50 MB default.
- [[components/page-buffer]]: documented `data_buffer_size` default = 32,768 × `db_page_size`; statdump exposes `Num_data_page_lru1/lru2/lru3` + victim candidates + private quota counters.
- [[components/log-manager]]: documented dual checkpoint triggers (`checkpoint_interval` 6 min + `checkpoint_every_size` ~156 MB); `;checkpoint` CSQL command (DBA only); `background_archiving=yes` (default) creates archives proactively.

## Key Insight

The admin guide is dense but well-structured: every `cubrid <verb>` has a reference subsection in `admin_utils.rst`. **Three operational gotchas dominate**: (1) HA users must use `cubrid heartbeat` not `cubrid server` (or risk bypassing failover), (2) backup is compressed by default since 11.2 so old `--no-compress` muscle memory matters, (3) the new `auto_restart_server` (11.4) means non-HA users finally get cub_master-driven recovery from OOM kills.
