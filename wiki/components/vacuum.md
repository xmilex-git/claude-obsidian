---
type: component
parent_module: "[[components/transaction|transaction]]"
path: "src/query/vacuum.c, vacuum.h"
status: active
purpose: "MVCC garbage collection daemon: reclaim dead heap rows, stale B-tree keys, and old version chains"
key_files:
  - "vacuum.h (VACUUM_WORKER, VACUUM_HEAP_OBJECT, vacuum_worker_state, logging macros)"
  - "vacuum.c (vacuum_process_log_block, vacuum master/worker thread logic)"
public_api:
  - "vacuum_initialize() / vacuum_finalize()"
  - "vacuum_process_log_block(thread_p, data) — process one block of MVCC log entries"
  - "vacuum_is_mvccid_vacuumed(mvccid) → bool"
  - "vacuum_notify_dropped_file(thread_p, ...)"
  - "vacuum_is_thread_vacuum(thread_p) → bool"
tags:
  - component
  - cubrid
  - vacuum
  - mvcc
  - gc
  - transaction
related:
  - "[[components/transaction|transaction]]"
  - "[[components/mvcc|mvcc]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/btree|btree]]"
  - "[[components/page-buffer|page-buffer]]"
created: 2026-04-23
updated: 2026-04-23
---

# Vacuum — MVCC Garbage Collection

> [!key-insight] Location note
> Despite being logically part of the transaction subsystem, `vacuum.c/h` lives in `src/query/`. The AGENTS.md explicitly states: "Fix vacuum/GC → `src/query/vacuum.c` (NOT in `src/transaction/`)." All other transaction-layer components are in `src/transaction/`.

Vacuum is CUBRID's MVCC garbage collector. Because CUBRID keeps old row versions as undo images in the WAL (not in heap as dead tuples), vacuum's primary job is:
1. Clear stale MVCC headers in heap records (remove `mvcc_ins_id`, `prev_version_lsa` once they are universally visible).
2. Remove heap records whose delete MVCCID is committed and older than all active snapshots.
3. Remove corresponding stale index keys from [[components/btree|btree]] indexes.

## Threading Model

Vacuum runs as dedicated daemon threads:

| Thread type | Count | Role |
|-------------|-------|------|
| `TT_VACUUM_MASTER` | 1 | Reads MVCC operation log; partitions work into blocks; dispatches jobs |
| `TT_VACUUM_WORKER` | 0–50 | Execute per-block vacuum jobs (configurable: `VACUUM_MAX_WORKER_COUNT = 50`) |

The master and workers are identified via `thread_p->type` checks:
```c
vacuum_is_thread_vacuum_master(thread_p)  // type == TT_VACUUM_MASTER
vacuum_is_thread_vacuum_worker(thread_p)  // type == TT_VACUUM_WORKER
```

## `VACUUM_WORKER` State Machine

Each worker runs through states:

```
VACUUM_WORKER_STATE_INACTIVE
    │ job dispatched
    ▼
VACUUM_WORKER_STATE_PROCESS_LOG   ← reads MVCC op log chain
    │ log processed
    ▼
VACUUM_WORKER_STATE_EXECUTE       ← physically cleans heap / btree pages
    │ job complete
    ▼
VACUUM_WORKER_STATE_INACTIVE
```

`vacuum_get_worker_state()` / `vacuum_set_worker_state()` are inlined accessors.

## `VACUUM_WORKER` Structure

```c
struct vacuum_worker {
  VACUUM_WORKER_STATE  state;
  INT32                drop_files_version;      /* detects dropped class events */
  struct log_zip      *log_zip_p;               /* LZ4 decompression buffer */
  VACUUM_HEAP_OBJECT  *heap_objects;            /* batch of OIDs to vacuum */
  int                  heap_objects_capacity;
  int                  n_heap_objects;
  char                *undo_data_buffer;        /* reusable undo image buffer */
  int                  undo_data_buffer_capacity;
  int                  private_lru_index;       /* page buffer private LRU */
  char                *prefetch_log_buffer;
  LOG_PAGEID           prefetch_first_pageid;
  LOG_PAGEID           prefetch_last_pageid;
  bool                 allocated_resources;
  int                  idx;                    /* -1 = master; ≥0 = worker index */
};
```

The `private_lru_index` gives each vacuum worker its own LRU partition in the page buffer, preventing vacuum from evicting hot data pages.

## MVCC Log Chain

The master follows the `LOG_VACUUM_INFO.prev_mvcc_op_log_lsa` chain in MVCC log records:

```
LOG_MVCC_UNDOREDO_DATA (newest)
  │ prev_mvcc_op_log_lsa
  ▼
LOG_MVCC_UNDOREDO_DATA
  │ prev_mvcc_op_log_lsa
  ▼
LOG_MVCC_UNDO_DATA (oldest MVCC op)
```

The master reads backward through this chain, collecting `VACUUM_HEAP_OBJECT` entries (VFID + OID). When a block of log pages is fully processed, the master creates a vacuum job and dispatches it to a worker.

## Vacuum Data File

`vacuum_data_vfid` (stored in `BOOT_DB_PARM`) is a dedicated file tracking:
- Which MVCC log blocks have been processed
- The oldest active MVCCID (used as the GC horizon)

`dropped_files_vfid` tracks classes/indexes that were dropped — vacuum workers check this to avoid cleaning files that no longer exist.

## Heap Vacuuming

A worker processes its batch of `VACUUM_HEAP_OBJECT` entries:
1. Fix the heap page containing the OID.
2. Call `mvcc_satisfies_vacuum(thread_p, rec_header, oldest_mvccid)`.
3. Based on result:
   - `VACUUM_RECORD_REMOVE` → physically delete the slot (mark slot free).
   - `VACUUM_RECORD_DELETE_INSID_PREV_VER` → clear `mvcc_ins_id` field and `prev_version_lsa`; set `MVCCID_ALL_VISIBLE`.
   - `VACUUM_RECORD_CANNOT_VACUUM` → skip.
4. Log the vacuum operation (`RVHF_*` recovery indexes).

## B-tree Vacuuming

For each vacuumed heap OID, any associated index keys are also cleaned. The vacuum worker calls into `btree_vacuum_*` functions (in [[components/btree|btree]]) which remove the MVCC-delete marker from the key's value list, or remove the key entirely if no live versions remain.

> [!warning] Vacuum vs active transactions
> Vacuum must check `oldest_mvccid` (the minimum MVCCID across all active snapshots) before reclaiming anything. A record deleted at MVCCID `X` can only be physically removed when `X < oldest_mvccid` — i.e., no active snapshot could possibly still see it. Incorrect `oldest_mvccid` computation leads to phantom reads or use-after-free in index pages.

## Dropped File Handling

When a class (table) is dropped:
1. `vacuum_notify_dropped_file()` records the drop in `dropped_files_vfid`.
2. Workers check `drop_files_version` at the start of each job.
3. If a worker encounters a VFID that matches a dropped file, it skips that OID.

`VACUUM_LOG_ADD_DROPPED_FILE_POSTPONE` / `UNDO` flags control whether the drop-notification log record is a postpone or undo record, ensuring correct behavior during rollback of a `DROP TABLE`.

## Logging and Diagnostics

`vacuum_er_log(level, msg, ...)` macros gate debug output behind `PRM_ID_ER_LOG_VACUUM` bitmask flags:

| Flag | Decimal | Covers |
|------|---------|--------|
| `VACUUM_ER_LOG_ERROR` | 1 | Errors |
| `VACUUM_ER_LOG_HEAP` | 16 | Heap vacuuming |
| `VACUUM_ER_LOG_BTREE` | 8 | B-tree vacuuming |
| `VACUUM_ER_LOG_WORKER` | 128 | Worker activity |
| `VACUUM_ER_LOG_MASTER` | 256 | Master activity |
| `VACUUM_ER_LOG_VERBOSE` | 0xFFFFFFFF | All |

## Related

- Parent: [[components/transaction|transaction]]
- [[components/mvcc|mvcc]] — `mvcc_satisfies_vacuum` is the core per-record GC predicate
- [[components/log-manager|log-manager]] — MVCC log records carry `LOG_VACUUM_INFO` chain pointers
- [[components/heap-file|heap-file]] — vacuum physically modifies heap slots and slots' MVCC headers
- [[components/btree|btree]] — vacuum removes stale MVCC markers from index key value lists
- [[components/page-buffer|page-buffer]] — each vacuum worker has a private LRU index to isolate its page access
- Source: [[sources/cubrid-src-transaction]]
