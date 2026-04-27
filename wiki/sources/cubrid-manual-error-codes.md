---
created: 2026-04-27
type: source
title: "CUBRID Manual — Error Code Catalogue"
source_path: "/home/cubrid/cubrid-manual/en/admin/error_log*.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - errors
  - reference
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-admin]]"
  - "[[sources/cubrid-manual-ha]]"
  - "[[components/error-manager]]"
---

# CUBRID Manual — Error Code Catalogue

**Ingested:** 2026-04-27
**Source files:** `admin/error_log.rst` (51) + `error_log_system.rst` (303, 43 codes), `error_log_admin.rst` (2194, 312 codes), `error_log_volume.rst` (631, 89 codes), `error_log_sql.rst` (1804, 257 codes), `error_log_transaction.rst` (383, 54 codes), `error_log_ha.rst` (261, 37 codes) — **792 total error codes**

## What This Is

Definitive reference for every CUBRID error code. **Codes are negative integers** (server convention). Each code has a category, a message template, and a contextual description.

This page is a **navigation index** — actual error lookups should hit the relevant RST file directly. Don't reproduce all 792 codes here.

## Code-Range Index

### `error_log_system.rst` (43 codes) — System / OS interface
- **-3** virtual memory exhaustion
- **-7..-16** disk volume format / mount / I/O
- **-78/-79** log-page I/O
- **-101..-105** volume info file
- **-111..-130** server boot/restart/memory
- **-313/-320** object corruption
- **-542/-543** disk-volume corruption
- **-708** temp-volume out-of-space
- **-779/-780/-792** system-time / connection-entry
- **-864/-879..-882** permission/lock issues
- **-980** thread-wait warning
- **-1185** page-buffer refix

### `error_log_admin.rst` (312 codes) — Database management (largest catalogue)
- Log path/prefix limits (`-83/-84`)
- DB version compatibility (`-86/-87`)
- Backup info (`-99`)
- createdb / addvoldb / restoredb / loaddb / checkdb / HA-config issues
- The bulk of operator-facing errors live here

### `error_log_sql.rst` (257 codes) — Query / SQL semantics
- Class/attribute lookup (`-64/-65/-202`)
- Type conversion (`-181/-182/-176`)
- NOT NULL constraint (`-205`)
- Many parser/optimizer/semantic errors
- This is where "your query is wrong" errors live

### `error_log_volume.rst` (89 codes) — Database file internals
- Slotted-page corruption (`-43..-46`)
- Heap creation (`-47`)
- Deleted/unknown OID (`-48/-49`)
- VFID lookup (`-38`)
- Unknown volid (`-35`)

### `error_log_transaction.rst` (54 codes) — Transaction / lock / MVCC
- **-72** unilateral abort
- **-73..-76** lock/latch timeout (object/class/instance/page) — **no deadlock**
- **-106..-108** 2PC
- **-440..-460** cursor / XASL
- **-550/-609/-640..-643** savepoints / rollback
- **-674** thread starvation
- **-836/-859** page-latch timeouts
- **-842..-856** lock-manager internals
- **-966..-968** **deadlock-induced** object/class/instance timeouts
- **-1021** **deadlock cycle detected** (the actual deadlock)
- **-1154..-1176** MVCC reevaluation / snapshot / latch promotion

### `error_log_ha.rst` (37 codes) — HA / replication
- **-898** generic replication
- **-970** HA-mode change
- **-986/-987** heartbeat lifecycle (started/stopped — NOTIFICATIONs, not errors)
- **-988..-990** node/process events
- **-1023..-1026** replication delay (paired threshold-cross / threshold-clear: -1025 trip / -1026 clear)
- **-1031..-1034** apply failures
- **-1035** applier OOM (`ha_log_applier_max_mem_size` exceeded)
- **-1036/-1037** applier/writer signal-shutdown
- **-1122/-1133/-1134** copylogdb/applylogdb
- **-1139..-1144** handshake errors
- **-1170** flush failure

### CAS error range (separate, transferred via CCI/JDBC)
- **-10001..-10200** — CAS-side errors. Catalogued in `control.rst` §CAS Error Code (lines ~2434-2507), not in `error_log_*.rst` files.
- **-10026 CAS_ER_MAX_PREPARED_STMT_COUNT_EXCEEDED** is a frequent example.

### CCI error range
- **-20001..-20999** — driver-side. `cci.rst:642-647`. `-20001 CCI_ER_DBMS` signals server error; the actual server `err_code` sits in `T_CCI_ERROR.err_code`.

### JDBC error range
- **-21001..-21999** — JDBC-side. `jdbc.rst:1073-1077`. `-21001..-21024` protocol/data-mapping. `-21101..-21141` JDBC API misuse (closed connection, unsupported method, etc.).

## How to Look Up an Error

```bash
# By code:
grep -rn "^-\?<NNNN>$\|^.. _<NNNN>:\|err_<NNNN>\|: -<NNNN>:" \
  /home/cubrid/cubrid-manual/en/admin/error_log_*.rst

# By keyword in message:
grep -rn -B 1 "<keyword>" /home/cubrid/cubrid-manual/en/admin/error_log_*.rst
```

For server source-side enum: `grep "ER_BTREE_UNIQUE_FAILED\|ER_LK_OBJECT_DL_TIMEOUT" ~/dev/cubrid/src/include/error_code.h` (per `control.rst:854-915`).

## Source-Code Anchors

- Error codes defined: `~/dev/cubrid/src/include/error_code.h` — `#define ER_* -NNN` macros
  - Examples: `ER_BTREE_UNIQUE_FAILED = -670`, `ER_UNIQUE_VIOLATION_WITHKEY = -886`, `ER_LK_OBJECT_DL_TIMEOUT_* = -966..-968`
- Messages: `$CUBRID/msg/<locale>/cubrid.msg` under `$set 5 MSGCAT_SET_ERROR`
- CAS errors: `~/dev/cubrid/src/broker/cas_error.h`

## Error Severity

`error_log_level` system parameter (default = NOTIFICATION):
- **NOTIFICATION** (most verbose) — informational events like `-986/-987` heartbeat lifecycle, `-1128/-1129` log-recovery start/finish
- **ERROR** — actual errors
- **WARNING** — recoverable conditions

ERROR < NOTIFICATION < WARNING in verbosity ordering (NOTIFICATION shows MORE than ERROR).

## Logging Locations

| Source | Log file |
|---|---|
| cub_server | `$CUBRID/log/server/<db>_<yyyymmdd>_<hhmi>.err` |
| cub_master | `$CUBRID/log/<host>.cub_master.err` |
| cub_pl | `$CUBRID/log/<db>_pl.err` |
| Broker CAS | `$CUBRID/log/broker/sql_log/`, `error_log/` |
| Utility | `$CUBRID/log/cubrid_utility.log` |

## Cross-References

- [[components/error-manager]] — error code propagation, `er_set()`, message catalog dispatch
- [[sources/cubrid-manual-admin]] — log file locations, troubleshooting workflows
- [[sources/cubrid-manual-ha]] — HA error catalogue interpretation
- [[sources/cubrid-manual-cci]] — CCI error code range and `T_CCI_ERROR` struct
- [[sources/cubrid-manual-jdbc]] — JDBC error code range

## Incidental Wiki Enhancements

- [[components/error-manager]]: documented the four error namespaces (server `<-9999`, transaction/lock subsets, CCI `-20001..-20999`, CAS `-10001..-10200`, JDBC `-21001..-21999`); the `ER_*` macros in `error_code.h`; messages live in `cubrid.msg` under `$set 5`.
- [[components/lock-manager]]: documented separate code ranges for lock timeout WITHOUT deadlock (`-73..-76`) vs deadlock-induced timeout (`-966..-968`) vs deadlock cycle (`-1021`).

## Key Insight

CUBRID has **multiple distinct error namespaces** — server (negative below -9999), CAS (-10001..-10200), CCI (-20001..-20999), JDBC (-21001..-21999). The same conceptual error (e.g., "server returned an error") gets re-encoded as it crosses each layer. The original server code is preserved inside `T_CCI_ERROR.err_code` (CCI) or as a chained exception (JDBC) — a debugger needs to peel off the layers. Lock-related codes are particularly granular: `-73..-76` are plain lock timeouts, `-836/-859` are latch timeouts, `-966..-968` are deadlock-induced, `-1021` is the deadlock detector firing.
