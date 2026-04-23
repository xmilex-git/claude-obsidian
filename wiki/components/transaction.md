---
type: component
parent_module: "[[modules/src|src]]"
path: "src/transaction/"
status: active
purpose: "MVCC, WAL, locking, deadlock detection, crash recovery, boot, vacuum coordination"
key_files:
  - "mvcc.c / mvcc.h (MVCC snapshot, visibility: mvcc_satisfies_snapshot)"
  - "lock_manager.c / lock_manager.h (LK_RES, LK_ENTRY, lock hierarchy)"
  - "wait_for_graph.c / wait_for_graph.h (WFG deadlock detection, DFS cycle finder)"
  - "log_append.cpp (WAL write path)"
  - "log_manager.c / log_manager.h (buffer, checkpoint, commit/abort)"
  - "log_record.hpp (LOG_RECORD_HEADER, LOG_RECTYPE enum, LSN)"
  - "log_recovery.c (ARIES: analysis → redo → undo)"
  - "log_tran_table.c (transaction descriptor table)"
  - "transaction_sr.c (server-side commit/abort/savepoint)"
  - "boot_sr.c (server boot sequence)"
  - "boot_cl.c (client-side boot/connect)"
public_api:
  - "mvcc_satisfies_snapshot(thread_p, rec_header, snapshot)"
  - "lock_object(thread_p, oid, class_oid, lock, cond_flag)"
  - "lock_unlock_all(thread_p)"
  - "log_append_undoredo_data(thread_p, rcvindex, addr, ...)"
  - "log_commit(thread_p, tran_index, retain_lock)"
  - "log_abort(thread_p, tran_index)"
  - "log_initialize(thread_p, db_fullname, logpath, prefix, ismedia_crash, r_args)"
tags:
  - component
  - cubrid
  - transaction
  - mvcc
  - wal
  - locking
  - recovery
  - server
related:
  - "[[modules/src|src]]"
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/btree|btree]]"
  - "[[components/heap-file|heap-file]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Error Handling Convention]]"
  - "[[Architecture Overview]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/transaction/` — Transactions, MVCC, WAL, Locking, Recovery

The transaction layer is the concurrency-control and durability backbone of the CUBRID engine. It spans MVCC snapshot management, a hierarchical lock manager with deadlock detection, ARIES-based write-ahead logging, crash recovery, server boot sequencing, and vacuum coordination. Almost every other server subsystem calls into this layer.

> [!key-insight] Scope note
> Vacuum (`vacuum.c`) physically lives in `src/query/` but is logically part of the transaction layer — it reads MVCC state exposed here and reclaims rows invisible to all active snapshots. The AGENTS.md explicitly says "Fix vacuum/GC → `src/query/vacuum.c` (NOT in this directory)."

## Architecture Overview

```
 Client request
       │
       ▼
 Transaction Table (log_tran_table.c)
 ┌─────────────────────────────────────────────────────────────────┐
 │  LOG_TDES per active transaction  (tran_index, MVCC_INFO, ...)  │
 └──────────┬──────────────────┬──────────────────┬───────────────┘
            │                  │                  │
            ▼                  ▼                  ▼
     MVCC subsystem      Lock Manager        WAL / Log Manager
     (mvcc.c/h)          (lock_manager.c)    (log_manager.c,
                               │              log_append.cpp)
                               │                  │
                         Deadlock detect     Log Page Buffer
                         (in lock_manager.c) (log_page_buffer.c)
                                                   │
                                              Disk (archive,
                                              active log vol)

                    Crash recovery (log_recovery.c)
                    ── Analysis → Redo → Undo (ARIES)

                    Boot sequencing (boot_sr.c)
                    ── subsystem init order, DB creation

                    Vacuum daemon (src/query/vacuum.c)
                    ── reads MVCC table → cleans dead rows
```

## Sub-Components

| Area | Files | Wiki page |
|------|-------|-----------|
| MVCC visibility | `mvcc.c`, `mvcc.h` | [[components/mvcc]] |
| Lock manager | `lock_manager.c`, `lock_manager.h` | [[components/lock-manager]] |
| Deadlock detection | inside `lock_manager.c` (active); `wait_for_graph.c` dead under `ENABLE_UNUSED_FUNCTION` | [[components/deadlock-detection]] |
| WAL / log manager | `log_manager.c/h`, `log_append.cpp`, `log_record.hpp` | [[components/log-manager]] |
| Crash recovery | `log_recovery.c` | [[components/recovery]] |
| Vacuum GC | `src/query/vacuum.c/h` | [[components/vacuum]] |
| Server boot | `boot_sr.c`, `boot_sr.h` | [[components/server-boot]] |
| Transaction table | `log_tran_table.c` | (inline — see log-manager) |
| Commit / abort | `transaction_sr.c` | (inline — see log-manager) |

## MVCC Model

Each heap record version carries an `MVCC_REC_HEADER` with:
- `mvcc_ins_id` — MVCCID of the inserting transaction (64-bit monotone, never reused)
- `mvcc_del_id` — MVCCID of the deleting transaction (`MVCCID_NULL` = live record)
- `prev_version_lsa` — LSA of the previous version in the log (version chain)

`mvcc_satisfies_snapshot()` decides visibility for a given `MVCC_SNAPSHOT`. The snapshot holds:
- `lowest_active_mvccid` — lower bound: anything below this is committed and visible
- `highest_completed_mvccid` — upper bound: anything at or above this is still running
- `m_active_mvccs` — bit array for the range in between

> [!key-insight] Three-range snapshot check
> `mvcc_is_id_in_snapshot()` uses three fast branches: (1) id below lowest → committed → not in snapshot, (2) id ≥ highest → still active → in snapshot, (3) otherwise consult the `m_active_mvccs` bitset. This avoids locking the transaction table for the common cases.

## Lock Hierarchy

```
Root class  (SCH-S / SCH-M)
  └── Class / Table  (IS / IX / S / X / SCH-S / SCH-M)
        └── Instance / Row  (S / X)
```

- Intent locks (`IS`, `IX`) on the table must be acquired before row-level locks.
- Schema locks (`SCH-S`, `SCH-M`) protect DDL operations.
- Lock escalation: when per-row lock count exceeds `LK_COMPOSITE_LOCK` threshold, escalates to table lock.
- Lock entries (`LK_ENTRY`) hang off a `LK_RES` (resource) via `holder` and `waiter` linked lists.
- The resource is identified by `LK_RES_KEY` = `{type, oid, class_oid}` and hashed in a lock-free hash map.

## WAL Protocol (ARIES-based)

1. Before modifying a page, append a log record via `log_append_undoredo_data()` (or variant).
2. Each record gets a `LOG_LSA` (log sequence number = page + offset in the log volume).
3. `pgbuf_flush_with_wal()` in [[components/page-buffer|page-buffer]] ensures the log is durable before the data page is written to disk.
4. Every `LOG_RECORD_HEADER` encodes `prev_tranlsa` (previous record for same transaction), `back_lsa`, `forw_lsa`, `trid`, and `type`.
5. MVCC operations use `LOG_REC_MVCC_UNDOREDO` which embeds an `MVCCID` and `LOG_VACUUM_INFO` so vacuum can trace the chain.

> [!key-insight] Compensation Log Records (CLR)
> `LOG_COMPENSATE` records mark that an undo action has been applied. During nested recovery (redo of a partial undo), CLRs prevent re-undoing already-undone records — a core ARIES safety property.

## Recovery Phases

`log_recovery.c` implements the three ARIES phases:

| Phase | What it does |
|-------|-------------|
| **Analysis** | Scan from last checkpoint; rebuild transaction table; determine redo start LSA and set of "losers" |
| **Redo** | Replay every log record from redo start LSA forward; reapplies changes of committed and in-flight transactions alike |
| **Undo** | Roll back all loser (uncommitted) transactions in reverse LSN order; writes CLR records |

Partially-written log pages (torn writes) are handled in `log_page_buffer.c` — the page is validated by magic number before use.

## Commit / Abort Flow

```
log_commit(thread_p, tran_index, retain_lock)
  └── log_commit_local(thread_p, tdes, ...)
        ├── log_do_postpone()        // run deferred redo ops
        ├── log_complete()           // write LOG_COMMIT record
        ├── lock_unlock_all()        // release all locks
        └── logtb_free_tran_index()  // return tran_index to pool
```

`transaction_sr.c` is the server-side entry for `xtran_server_commit()` / `xtran_server_abort()` called by the network handler.

## Server Boot Sequence

`boot_sr.c` initializes subsystems in strict dependency order:
1. Parameters, language, TZ support
2. `disk_manager` → `file_manager` → `page_buffer`
3. `log_initialize()` — opens/creates log volume; triggers crash recovery if needed
4. Lock manager, catalog, heap, session, vacuum
5. PL engine JNI bridge (SERVER_MODE)

`BOOT_DB_PARM` (stored in a root heap page) carries persistent metadata: tracker VFID, root class HFID, catalog CTID, vacuum data VFID, TDE key info HFID.

> [!warning] Boot ordering is fragile
> Subsystems must be initialized in exact order — e.g. page buffer before log manager, log manager before lock manager. Adding a new early-boot subsystem that calls into log or lock before they are ready will deadlock or assert.

## Build Mode

All files are `SERVER_MODE` + `SA_MODE` only (client stub in `boot_cl.c` for `CS_MODE`). The lock manager's `LK_ENTRY` struct is empty (`int dummy`) in non-server builds, making it safe to link without the full implementation.

## Common Modification Points

| Task | File | Entry point |
|------|------|-------------|
| Fix MVCC visibility | `mvcc.c` | `mvcc_satisfies_snapshot()` |
| Fix deadlock detection | `lock_manager.c`, `wait_for_graph.c` | `lock_detect_deadlock()` |
| Fix WAL write | `log_append.cpp` | `log_append_undoredo_data()` |
| Fix crash recovery | `log_recovery.c` | `log_rv_analysis()`, `log_rv_redo_record()`, `log_rv_undo_record()` |
| Fix checkpoint | `log_manager.c` | `logpb_checkpoint()` |
| Fix commit/abort | `transaction_sr.c` | `xtran_server_commit()` |
| Fix vacuum | `src/query/vacuum.c` | `vacuum_process_log_block()` |
| Fix boot/startup | `boot_sr.c` | `boot_restart_server()` |

## Gotchas

- `lock_manager.c` and `log_recovery.c` are each 15K+ lines — intentional, not tech debt.
- MVCC IDs are 64-bit monotonically increasing integers; never reused even across restarts.
- Vacuum runs as a daemon (`TT_VACUUM_MASTER` + up to 50 `TT_VACUUM_WORKER` threads) and must race-safely read the MVCC table.
- Log record types include `LOG_SUPPLEMENTAL_INFO` (type 52) for CDC (Change Data Capture) support — not a core ARIES record.
- `wait_for_graph.c` is compiled only when `ENABLE_UNUSED_FUNCTION` is defined; the active deadlock detector is embedded in `lock_manager.c`.

## Related

- Parent: [[modules/src|src]]
- [[components/storage|storage]] — pages being protected; `pgbuf_flush_with_wal` enforces WAL ordering
- [[components/page-buffer|page-buffer]] — page LSA checked at flush time against log
- [[components/btree|btree]] — uses 18 `btree_op_purpose` values for MVCC operations on index keys
- [[components/heap-file|heap-file]] — MVCC-aware heap scan, `MVCC_REC_HEADER` embedded in record
- [[Memory Management Conventions]]
- [[Build Modes (SERVER SA CS)]]
- [[Error Handling Convention]]
- Source: [[sources/cubrid-src-transaction]]
