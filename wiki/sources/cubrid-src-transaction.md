---
type: source
title: "CUBRID src/transaction/ — Transaction Layer Source"
source_type: codebase
path: "src/transaction/"
date_ingested: 2026-04-23
status: complete
tags:
  - cubrid
  - source
  - transaction
  - mvcc
  - wal
  - locking
  - recovery
related:
  - "[[components/transaction|transaction]]"
  - "[[components/mvcc|mvcc]]"
  - "[[components/lock-manager|lock-manager]]"
  - "[[components/deadlock-detection|deadlock-detection]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/recovery|recovery]]"
  - "[[components/vacuum|vacuum]]"
  - "[[components/server-boot|server-boot]]"
---

# Source: `src/transaction/` — CUBRID Transaction Layer

## What Was Read

| File | Purpose |
|------|---------|
| `AGENTS.md` | Directory-level orientation; file-to-task mapping |
| `mvcc.h` | `MVCC_REC_HEADER`, `MVCC_SNAPSHOT`, `MVCC_INFO` structs; visibility enums |
| `mvcc.c` (head + `mvcc_satisfies_snapshot`) | Snapshot check implementation; `mvcc_is_id_in_snapshot` fast-path |
| `lock_manager.h` | `LK_RES`, `LK_ENTRY`, lock result codes, public API |
| `lock_manager.c` (head) | Hash macros, thread-wait macros, `LK_ISYOUNGER` |
| `wait_for_graph.c` (head) | `WFG_NODE`, `WFG_EDGE`, `WFG_STACK` DFS structures |
| `log_record.hpp` (first 230 lines) | `LOG_RECORD_HEADER`, `log_rectype` enum (52 types), MVCC log records |
| `log_manager.h` (first 160 lines) | `log_append_*` family API, `log_commit`, `log_abort`, `log_initialize` |
| `boot_sr.c` (head) | Include list, `BOOT_DB_PARM`, `boot_Server_status` |
| `src/query/vacuum.h` | `VACUUM_WORKER`, `VACUUM_WORKER_STATE`, logging macros, thread-type checks |

## Key Findings

### MVCC
- Old versions are kept in the WAL, not in heap (unlike PostgreSQL). Visibility checks that return `TOO_NEW_FOR_SNAPSHOT` require log I/O to follow `prev_version_lsa`.
- Three-range snapshot check in `mvcc_is_id_in_snapshot` avoids transaction-table lookup for most records.
- `MVCCID_ALL_VISIBLE` is a sentinel meaning vacuum has cleared the insert ID — the record is visible to all.
- `mvcc_is_mvcc_disabled_class()` bypasses MVCC for system catalog tables.

### Locking
- `wait_for_graph.c` is compiled only under `ENABLE_UNUSED_FUNCTION`; the active deadlock detector is embedded in `lock_manager.c`.
- Deadlock victim = youngest transaction (`LK_ISYOUNGER` = higher tran_index).
- Composite lock API (`LK_COMPOSITE_LOCK`) for batch DML lock escalation.
- `LK_ENTRY` struct is an empty `{int dummy}` in non-server builds — safe to link in CS_MODE.

### WAL / Log
- 52 `log_rectype` values; types 46–50 are MVCC-specific or atomic-sysop markers.
- `LOG_SUPPLEMENTAL_INFO` (type 52) is for CDC and does not affect ARIES recovery.
- `LOG_VACUUM_INFO.prev_mvcc_op_log_lsa` threads MVCC log records into a chain for the vacuum master to walk.
- Log records support crumb (scatter-gather) writes to avoid data copies.

### Recovery
- Standard ARIES: Analysis → Redo → Undo.
- `LOG_SYSOP_ATOMIC_START` triggers immediate rollback of incomplete atomic ops (B-tree splits) before postpones run.
- 2PC in-doubt transactions are left in prepared state; not unilaterally resolved.

### Boot
- `BOOT_DB_PARM` carries vacuum VFIDs and TDE key info HFID alongside the classic file topology fields.
- Init order is strict: disk → file → page-buffer → log (+ recovery) → lock → catalog → heap → session → vacuum.
- `boot_Enabled_flush_daemons` gates background flush after server is fully up.

### Vacuum
- Lives in `src/query/` not `src/transaction/` — explicit gotcha from AGENTS.md.
- Max 50 workers; each has a private LRU index in the page buffer.
- Worker state machine: INACTIVE → PROCESS_LOG → EXECUTE → INACTIVE.

## Pages Created

- [[components/transaction|transaction]] — upgraded from stub
- [[components/mvcc|mvcc]] — new
- [[components/lock-manager|lock-manager]] — new
- [[components/deadlock-detection|deadlock-detection]] — new
- [[components/log-manager|log-manager]] — new
- [[components/recovery|recovery]] — new
- [[components/vacuum|vacuum]] — new
- [[components/server-boot|server-boot]] — new
