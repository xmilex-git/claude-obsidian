---
type: component
parent_module: "[[components/transaction|transaction]]"
path: "src/transaction/log_manager.c, log_append.cpp, log_record.hpp, log_page_buffer.c"
status: active
purpose: "ARIES-based Write-Ahead Logging: append, buffer management, checkpoint, commit/abort"
key_files:
  - "log_manager.h / log_manager.c (buffer mgmt, checkpoint, commit/abort, log_initialize)"
  - "log_append.cpp (write path: log_append_undoredo_data and variants)"
  - "log_record.hpp (LOG_RECORD_HEADER, log_rectype enum, LOG_REC_UNDOREDO, LOG_REC_MVCC_UNDOREDO)"
  - "log_page_buffer.c (log page I/O, archive management)"
  - "log_tran_table.c (LOG_TDES transaction descriptor table)"
  - "log_compress.c (LZ4 compression of log records)"
  - "log_comm.c (replication log communication)"
  - "log_lsa.hpp (LOG_LSA struct)"
public_api:
  - "log_initialize(thread_p, db_fullname, logpath, prefix, ismedia_crash, r_args)"
  - "log_append_undoredo_data(thread_p, rcvindex, addr, undo_len, redo_len, undo_data, redo_data)"
  - "log_append_undo_data(thread_p, rcvindex, addr, length, data)"
  - "log_append_redo_data(thread_p, rcvindex, addr, length, data)"
  - "log_append_compensate(thread_p, rcvindex, vpid, offset, pgptr, length, data, tdes)"
  - "log_append_savepoint(thread_p, savept_name) → LOG_LSA*"
  - "log_commit(thread_p, tran_index, retain_lock) → TRAN_STATE"
  - "log_abort(thread_p, tran_index) → TRAN_STATE"
  - "log_abort_partial(thread_p, savepoint_name, savept_lsa) → TRAN_STATE"
  - "log_get_append_lsa() → LOG_LSA*"
  - "logpb_checkpoint() (internal)"
tags:
  - component
  - cubrid
  - wal
  - logging
  - transaction
  - recovery
related:
  - "[[components/transaction|transaction]]"
  - "[[components/recovery|recovery]]"
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/vacuum|vacuum]]"
  - "[[components/server-boot|server-boot]]"
created: 2026-04-23
updated: 2026-04-23
---

# Log Manager — Write-Ahead Logging

CUBRID's log subsystem implements ARIES (Algorithm for Recovery and Isolation Exploiting Semantics). All durable changes must be logged before the modified data page can be written to disk. The implementation spans several files; `log_manager.c` orchestrates the high-level lifecycle while `log_append.cpp` owns the hot write path.

## Log Structure

### Physical layout

```
Active log volume (cubrid.lgar)
  ┌─────────────────────────────────────────────────────┐
  │ Log header page (volid=LOG_DBLOG_ACTIVE_VOLID)      │
  │   chkpt_lsa, eof_lsa, append_lsa, db_creation, ... │
  ├─────────────────────────────────────────────────────┤
  │ Log page 1   [LOG_PAGE_SIZE bytes]                  │
  │   LOG_RECORD_HEADER | body | LOG_RECORD_HEADER | …  │
  ├─────────────────────────────────────────────────────┤
  │ Log page 2 …                                        │
  └─────────────────────────────────────────────────────┘

Archive volumes (cubrid.lgar000, lgar001, …)
  Rotated when active volume fills; old pages read during recovery.
```

### `LOG_LSA` — Log Sequence Number

```c
// from log_lsa.hpp
struct log_lsa {
  INT64 pageid;  /* log page index */
  INT16 offset;  /* byte offset within that page */
};
```

`NULL_LSA` (`{-1, -1}`) means "no LSA." Every modified data page carries an embedded `page LSA` set via `pgbuf_set_lsa()`.

## Key Structures (`log_record.hpp`)

### `LOG_RECORD_HEADER`

```c
struct log_rec_header {
  LOG_LSA   prev_tranlsa;  /* previous log record for same transaction */
  LOG_LSA   back_lsa;      /* backward link (for reverse scan) */
  LOG_LSA   forw_lsa;      /* forward link */
  TRANID    trid;          /* transaction identifier */
  LOG_RECTYPE type;        /* record type */
};
```

Every log record starts with this header. The `prev_tranlsa` chain allows walking a transaction's records backward (used during undo).

### `log_rectype` Enum — Selected important types

| Value | Name | Purpose |
|-------|------|---------|
| 2 | `LOG_UNDOREDO_DATA` | Standard undo+redo record |
| 3 | `LOG_UNDO_DATA` | Undo-only |
| 4 | `LOG_REDO_DATA` | Redo-only |
| 8 | `LOG_COMPENSATE` | CLR — marks completed undo action |
| 17 | `LOG_COMMIT` | Transaction committed |
| 20 | `LOG_SYSOP_END` | End of system operation (4 sub-types) |
| 22 | `LOG_ABORT` | Transaction aborted |
| 25 | `LOG_START_CHKPT` | Checkpoint start marker |
| 26 | `LOG_END_CHKPT` | Checkpoint data |
| 27 | `LOG_SAVEPOINT` | User savepoint |
| 46 | `LOG_MVCC_UNDOREDO_DATA` | MVCC undoredo (includes MVCCID + vacuum info) |
| 50 | `LOG_SYSOP_ATOMIC_START` | Atomic op marker (rollback-immediately on crash) |
| 52 | `LOG_SUPPLEMENTAL_INFO` | CDC supplemental log |

### MVCC log records

```c
struct log_rec_mvcc_undoredo {
  LOG_REC_UNDOREDO  undoredo;   /* standard undoredo fields */
  MVCCID            mvccid;     /* inserting/deleting transaction's MVCC ID */
  LOG_VACUUM_INFO   vacuum_info; /* prev_mvcc_op_log_lsa + VFID for vacuum chain */
};
```

The `LOG_VACUUM_INFO.prev_mvcc_op_log_lsa` links MVCC operations into a chain that the [[components/vacuum|vacuum]] daemon follows backward to find dead versions.

## Log Append Path

```
Caller: log_append_undoredo_data(thread_p, rcvindex, addr, undo_len, redo_len, ...)
  │
  ▼ log_append.cpp
  log_append_undoredo_data_internal()
    ├── Acquire log_Gl.append.log_write_mutex (or use lock-free path)
    ├── Reserve space in log page buffer (log_Gl.append.log_pgptr)
    │     If page full → flush current page, advance to next
    ├── Write LOG_RECORD_HEADER + LOG_REC_UNDOREDO body + undo/redo data
    ├── Update tdes->tail_lsa (last record for this transaction)
    ├── Update log_Gl.append_lsa
    └── Release mutex
```

Log records can be written as **crumbs** (scatter-gather: `LOG_CRUMB[]`) to avoid copying large data blocks. The `log_append_undoredo_crumbs()` variant accepts crumb arrays.

> [!key-insight] Log compression
> `log_compress.c` uses LZ4 to compress undo/redo data when it exceeds `LOG_MIN_COMPRESS_SIZE`. Compressed records are stored with a `LOG_ZIP` prefix; recovery decompresses before applying.

## WAL Enforcement

The page buffer enforces WAL ordering in `pgbuf_flush_with_wal()`:

```
pgbuf_flush_with_wal(thread_p, pgptr)
  ├── Get page's current LSA (pgbuf_get_lsa)
  ├── log_flush_if_needed(page_lsa)  ← ensures log is at least as durable as the page
  └── Then flush page to disk (or DWB)
```

This guarantees that a data page never reaches disk before its log records do. See [[components/storage|storage]] and [[components/page-buffer|page-buffer]] for the storage side.

## Checkpoint

`logpb_checkpoint()` (called periodically by the checkpoint daemon):
1. Writes `LOG_START_CHKPT` record at current append LSA.
2. Flushes all dirty data pages up to the checkpoint LSA.
3. Writes `LOG_END_CHKPT` with: active transaction list, dirty page list, checkpoint LSA.
4. Updates log header's `chkpt_lsa`.
5. Archives old log pages that are no longer needed for recovery.

After checkpoint, recovery only needs to re-read from `chkpt_lsa` forward.

## Transaction Lifecycle in the Log

```
log_initialize()              ← called by boot_sr.c
  │
  ├── Crash recovery if needed (delegates to log_recovery.c)
  │
  ├── [Normal operation]
  │     log_append_*()         ← called by heap, btree, etc.
  │     log_commit()
  │       ├── log_do_postpone()
  │       ├── log_complete() → write LOG_COMMIT
  │       └── lock_unlock_all()
  │     log_abort()
  │       ├── log_rv_undo_rec() per undo record (reverse order)
  │       └── write LOG_ABORT
  │
  └── log_final()              ← called by boot finalize
```

## System Operations (`LOG_SYSOP`)

> [!info] Dedicated component page
> Full coverage of the `log_sysop_*()` family (18 functions, lifecycle, end-type matrix, postpone interaction, atomic-sysop recovery semantics): [[components/log-sysop]].


System operations allow grouping multiple page changes into a logical atomic unit that:
- Commits independently of the user transaction (`LOG_SYSOP_END_COMMIT`) — used for catalog updates
- Holds undo data for logical rollback (`LOG_SYSOP_END_LOGICAL_UNDO`)
- Compensates a prior change (`LOG_SYSOP_END_LOGICAL_COMPENSATE`)
- Runs a postpone record (`LOG_SYSOP_END_LOGICAL_RUN_POSTPONE`)

`LOG_SYSOP_ATOMIC_START` (type 50) marks operations that must be immediately rolled back if recovery encounters them incomplete during the redo phase — preventing partial index splits from persisting.

## Log Communication (Replication)

`log_comm.c` / `log_comm.h` provide the protocol for streaming log records to HA standby nodes. `LOG_REPLICATION_DATA` (type 39) and `LOG_REPLICATION_STATEMENT` (type 40) records carry DML and DDL changes respectively.

## Supplemental Logging (CDC)

`LOG_SUPPLEMENTAL_INFO` (type 52) carries before/after images and transaction metadata for the Change Data Capture (`cubrid_log` API in `src/api/`). This is additive to normal ARIES logging and does not affect recovery correctness.

## Related

- Parent: [[components/transaction|transaction]]
- [[components/recovery|recovery]] — consumes log records during crash recovery
- [[components/storage|storage]] — WAL ordering enforced via `pgbuf_flush_with_wal`
- [[components/page-buffer|page-buffer]] — page LSA checked at flush time
- [[components/vacuum|vacuum]] — follows `LOG_VACUUM_INFO.prev_mvcc_op_log_lsa` chain
- [[components/server-boot|server-boot]] — calls `log_initialize()` early in boot sequence
- Source: [[sources/cubrid-src-transaction]]
