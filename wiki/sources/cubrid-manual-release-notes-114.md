---
created: 2026-04-27
type: source
title: "CUBRID 11.4 Release Notes"
source_path: "/home/cubrid/cubrid-manual/en/release_note/release_note_latest_ver.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - release-notes
  - version-11.4
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-plcsql]]"
  - "[[sources/cubrid-manual-sql-tuning-parallel]]"
  - "[[sources/cubrid-manual-sql-foundation]]"
  - "[[sources/cubrid-manual-admin]]"
---

# CUBRID 11.4 Release Notes

**Ingested:** 2026-04-27
**Source file:** `/home/cubrid/cubrid-manual/en/release_note/release_note_latest_ver.rst` (1235 lines)
**Build:** **11.4.0.1778**

## Headline Features (9 items)

1. **PL/CSQL** — Oracle PL/SQL compatibility (the headline feature)
2. **HASH JOIN** — opt-in via `/*+ USE_HASH */`
3. **Optimizer + index processing improvements**
4. **Parallel REDO recovery** — multi-threaded log apply during recovery
5. **Result caching expanded** — CTE + uncorrelated subquery
6. **Improved data dump performance** — multi-threaded `unloaddb`
7. **Memory monitoring** (`cubrid memmon`)
8. **Enhanced access control** (`ACCESS_CONTROL_DEFAULT_POLICY`, per-broker ACL)
9. **Improved backup/recovery convenience** (`restore_to_newdb.sh`, `--separate-keys`)

## New Features

### SQL
- **PL/CSQL** — see [[sources/cubrid-manual-plcsql]]
- **HASH JOIN** — `/*+ USE_HASH */`, `/*+ NO_USE_HASH */`
- **`ALTER SERIAL serial_name OWNER TO user_name`** — was `call change_serial_owner()` method. DBA / DBA-group only.
- **`ALTER USER ... ADD MEMBERS / DROP MEMBERS`** — was `call add_member()` / `call drop_member()`. Simplifies HA group mgmt.
- **Extended Query Cache** — works in CTEs and uncorrelated subqueries; trace shows `RESULT CACHE (reference count: N)`.
- **`/*+ LEADING(t1 t2) */`** — finer than ORDERED for join order. Ignored when ORDERED present, when join graph dependencies prevent early joining.

### Utility
- **`cubrid plandump -s <plan_id>`** — delete specific plan (was -d for all)
- **`loadjava -j` / `--jni`** — separate JNI registration for Java SP that calls `System.load()` of native libs. Required as of 11.4 for JNI-based Java SP.
- **`cubrid memmon <db>`** — heap memory profile from `cub_server`. Requires `enable_memory_monitoring=yes` + restart. Per-source-line allocation tracking.
- **`diagdb -d 9 -n <class>`** — dump heap data for a specific class (table). Partitioned tables: all partitions; sub-partitioned: only the named partition.

### Broker / CAS / CMS
- **`NET_BUF_SIZE = 16K | 32K | 48K | 64K`** — configurable Broker→client send buffer.
- **`ACCESS_CONTROL_DEFAULT_POLICY = DENY | ALLOW`** — default action for brokers not in `ACCESS_CONTROL_FILE`. Default DENY.

### Others
- **`auto_restart_server`** — non-HA cub_server auto-restart on abnormal termination (OOM, segfault). Disabled if too-frequent crashes within 120 s or repeated start failures. Does not auto-restart on normal shutdown.
- **`restore_to_newdb.sh`** — restore backup as a new DB name; auto-registers in `databases.txt`.

## Specification Changes (Breaking)

### SQL
- **CHAR max chars dropped from 268,435,456 → 2048.** Migration: oversize CHAR → VARCHAR.
- **LOB locator path: relative not absolute** — easier directory moves; existing LOB files must be moved to new dir if changed.
- **Analytic functions in UPDATE JOIN: error.** Two+ table UPDATE with analytic in SET = error in 11.4.
- **`for update` on system tables/views: error.**
- **View creation: type checks deferred to runtime** (Oracle compat).
- **NULL allowed in view SELECT clause** (Oracle compat).
- **AUTO_INCREMENT + DEFAULT together: error** in CREATE TABLE / ALTER COLUMN.
- **`ALTER INDEX ... REBUILD` with new columns: error.**
- **ROWNUM exceeding NUMERIC range: error** (was silent overflow).

### Utility
- **CSQL recognizes `;<session>` mid-CREATE-PROCEDURE/BODY** (interactive mode) — was previously confused.
- **`cubrid plandump -d`** suppresses plan info output before deletion.

### HA
- **`ha_mode=on` with `myhost` in `ha_node_list`: error.** Previously auto-switched to replica mode.

### Others
- **`db_serial.att_name → attr_name`** column rename.

## Improvements (Performance + Quality of Life)

### SQL
- **Caching for correlated subqueries** — repeated identical condition values hit cache.
- **Sort-Limit with bind variables / expressions** in LIMIT — now optimized.

### PL / Java SP
- **Auto-restart of JavaSP server** on JNI segfault.
- **Multi-charset string handling** in JavaSP (euckr, utf8 — process as byte arrays).

### Index
- **Removed length limit on filter-index WHERE clause.**
- **midxkey.buf size optimization** for multi-column indexes — direct OFFSET reference, no recalculation. Improves binary search, key filtering, DML.
- **Range-Scan key processing optimized** — fewer upper_key comparisons, fewer column-ID comparisons, common-prefix info added to leaf node headers.

### Optimizer (extensive)
- **Sampling pages 1000 → 5000** for stats accuracy.
- **NDV (number of distinct values) collection** improved.
- **Reduced rule-based optimization** — more cost-based.
- **Index Filter Scan selectivity** in I/O cost.
- **NOT LIKE selectivity** added.
- **Function-based index selectivity** added.
- **NDV duplicate weighting** (>1% sample dup → reweight).
- **`SSCAN_DEFAULT_CARD`** — prevents inefficient NL JOIN plans on tiny cardinality.
- **LIMIT cost/cardinality** reflected in plans.
- **Eliminated redundant join conditions** more aggressively.
- **Removed unnecessary INNER JOIN** — additional join elimination.
- **Trace `fetch_time` field** added.
- **Index Skip Scan auto-selected** (no `index_ss` hint needed).
- **Optimizer prefers better index over Primary Key** when cheaper.
- **Stored Procedure execution plans**: index scans now used; unnecessary joins eliminated; result caching in correlated subqueries.
- **Concurrency**: unnecessary X-LOCKs released after all conditions evaluated.
- **Index scan in `LIKE` queries with JavaSP functions** improved.
- **Range Index Scan for `<=`/`>=` on function indexes** improved.

### Utility
- **`unloaddb` even with no tables/views** — extracts user/serial/sp/server/synonym/grant.
- **`unloaddb` multi-threaded** — `--thread-count` (0-127), `--enhanced-estimates` (verbose).
- **CSQL env var rename**: `EDITOR` → `CUBRID_CSQL_EDITOR`, `SHELL` → `CUBRID_CSQL_SHELL`, `FORMATTER` → `CUBRID_CSQL_FORMATTER`. Old names still accepted.
- **`cubrid spacedb`** improved: overflow pages classified as INDEX/HEAP (not SYSTEM); ADMIN_LOG_FILE display added; absolute paths shown.

### Broker / CMS
- **TLS v1.2 support** for CMS clients.
- **Zombie process prevention** when broker/CAS terminated through CMS.
- **CMS `getlogfileinfo()`** — returns SQL log file info only once.
- **CMS `ha_status()`** — displays replica node status in master/slave/replica configs.

### HA
- **Clear error messages** for incorrect `ha_node_list` / `ha_replica_list`.
- **Clear messages** for failover/failback events.
- **Prevent incorrect data-mismatch messages** when `restoreslave` runs on slave.

### Others
- **`cubrid_host.conf`** improvements: "0.0.0.0 your_hostname" entry; case-insensitive hostnames; IP+hostname validation.
- **DDL audit log** improvements: 'DB Name' added to CAS-generated DDL audit logs (multi-DB env disambiguation); ABORT log even without commit/rollback; multiple-DDL-in-one-txn handling under SetAutocommit(false).
- **REDO recovery parallel threads** (page-by-page parallel apply where no synchronization required).
- **Simplified deadlock event log** — only contributing locks shown.
- **`time_format()` / `date_format()`** speedup — removed unnecessary string ops.
- **Query perf with TRACE on** — reduced trace-stat-gathering cost.
- **String length checks removed** where unnecessary.
- **Lock memory bounded by `lock_escalation`** — prevents runaway lock memory.
- **Invalid `optimization_level`** now raises error (was silent).
- **`SHOW VOLUME HEADER`, `SHOW LOG HEADER`, `SHOW ARCHIVE LOG HEADER`** — added Creation_time field.
- **`diagdb`** — volume creation times.
- **`applyinfo`** — added volume creation time in log volume header.
- **`SHOW INDEX CAPACITY` + `diagdb`** — fence key info now separately displayed.

## Bug Fixes

The release notes have a Bug Fixes section listing dozens of fixes by category (SQL / PL / Index / Optimizer / Utility / Broker/CAS/CMS / HA / Others). Out of scope for this catalog — read directly when chasing a specific regression.

## Driver Compatibility

CUBRID 11.4 ensures compatibility with earlier driver versions, but new features need new drivers.

## Cross-References

- [[sources/cubrid-manual-plcsql]] — PL/CSQL language reference
- [[sources/cubrid-manual-sql-tuning-parallel]] — HASH JOIN, LEADING hint, expanded query cache
- [[sources/cubrid-manual-sql-foundation]] — SQL spec changes
- [[sources/cubrid-manual-sql-ddl]] — ALTER USER / ALTER SERIAL new SQL
- [[sources/cubrid-manual-admin]] — `cubrid memmon`, `cubrid plandump -s`, `restore_to_newdb.sh`, `auto_restart_server`
- [[sources/cubrid-manual-config-params]] — new params (NET_BUF_SIZE, ACCESS_CONTROL_DEFAULT_POLICY, auto_restart_server, enable_memory_monitoring)

## Incidental Wiki Enhancements

- Captured under each cluster page (sql-foundation, sql-ddl, sql-tuning-parallel, admin, config-params, plcsql, pl).

## Key Insight

CUBRID 11.4 is a **substantial release**: PL/CSQL (Oracle compat language) is the marquee, but the optimizer wave (HASH JOIN, LEADING, NDV improvements, function-based-index selectivity, SP plan revamp) is just as impactful for existing workloads. The **breaking specs** are the migration trap — CHAR max chars 268M → 2048, view creation type checks deferred, AUTO_INCREMENT+DEFAULT illegal, `db_serial.att_name → attr_name`. The **memmon utility + `enable_memory_monitoring`** is the first time CUBRID has shipped per-source-line heap profiling.
