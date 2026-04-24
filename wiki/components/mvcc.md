---
type: component
parent_module: "[[components/transaction|transaction]]"
path: "src/transaction/mvcc.c, mvcc.h"
status: active
purpose: "MVCC snapshot management and record visibility predicates"
key_files:
  - "mvcc.h (MVCC_REC_HEADER, MVCC_SNAPSHOT, MVCC_INFO structs; visibility enums)"
  - "mvcc.c (mvcc_satisfies_snapshot, mvcc_satisfies_delete, mvcc_satisfies_vacuum)"
  - "mvcc_active_tran.hpp (m_active_mvccs bitset)"
public_api:
  - "mvcc_satisfies_snapshot(thread_p, rec_header, snapshot) → MVCC_SATISFIES_SNAPSHOT_RESULT"
  - "mvcc_satisfies_delete(thread_p, rec_header) → MVCC_SATISFIES_DELETE_RESULT"
  - "mvcc_satisfies_vacuum(thread_p, rec_header, oldest_mvccid) → MVCC_SATISFIES_VACUUM_RESULT"
  - "mvcc_satisfies_dirty(thread_p, rec_header, snapshot) → MVCC_SATISFIES_SNAPSHOT_RESULT"
  - "mvcc_is_mvcc_disabled_class(class_oid) → bool"
tags:
  - component
  - cubrid
  - mvcc
  - transaction
  - visibility
related:
  - "[[components/transaction|transaction]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/btree|btree]]"
  - "[[components/vacuum|vacuum]]"
  - "[[components/log-manager|log-manager]]"
created: 2026-04-23
updated: 2026-04-23
---

# MVCC — Multi-Version Concurrency Control

CUBRID's MVCC implementation lives in `src/transaction/mvcc.c/h`. It answers one core question for every heap record version: **is this version visible to the current transaction's snapshot?**

## Key Structures

### `MVCC_REC_HEADER` — Per-record version metadata

```c
struct mvcc_rec_header {
  INT32 mvcc_flag : 8;       /* OR_MVCC_FLAG_VALID_INSID / VALID_DELID / VALID_PREV_VERSION */
  INT32 repid    : 24;       /* representation id (schema version) */
  int   chn;                 /* cache coherency number */
  MVCCID mvcc_ins_id;        /* inserting transaction's MVCC ID */
  MVCCID mvcc_del_id;        /* deleting transaction's MVCC ID (MVCCID_NULL = live) */
  LOG_LSA prev_version_lsa;  /* LSA of previous version in undo log */
};
```

The `mvcc_flag` bits control which optional fields are present on disk (flags: `OR_MVCC_FLAG_VALID_INSID`, `OR_MVCC_FLAG_VALID_DELID`, `OR_MVCC_FLAG_VALID_PREV_VERSION`). Vacuum clears `VALID_INSID` once the inserter has committed and no snapshot can see it as in-progress, writing `MVCCID_ALL_VISIBLE` as a sentinel.

### `MVCC_SNAPSHOT` — Per-query visibility boundary

```c
struct mvcc_snapshot {
  MVCCID lowest_active_mvccid;     /* all IDs below this are committed */
  MVCCID highest_completed_mvccid; /* all IDs >= this are still running */
  mvcc_active_tran m_active_mvccs; /* bitset for the uncertain range */
  MVCC_SNAPSHOT_FUNC snapshot_fnc; /* pointer to the chosen visibility function */
  bool valid;
};
```

### `MVCC_INFO` — Per-transaction MVCC state (embedded in `LOG_TDES`)

```c
struct mvcc_info {
  MVCC_SNAPSHOT snapshot;
  MVCCID id;                              /* this transaction's MVCC ID */
  MVCCID recent_snapshot_lowest_active_mvccid; /* fast lower-bound cache */
  std::vector<MVCCID> sub_ids;            /* sub-transaction IDs */
  bool is_sub_active;
};
```

## Visibility Predicates

### `mvcc_satisfies_snapshot` — Main read path

Returns `MVCC_SATISFIES_SNAPSHOT_RESULT`:

| Result | Meaning |
|--------|---------|
| `SNAPSHOT_SATISFIED` | Record is visible to this snapshot |
| `TOO_NEW_FOR_SNAPSHOT` | Inserter was still active at snapshot time; caller must check `prev_version_lsa` |
| `TOO_OLD_FOR_SNAPSHOT` | Record was deleted before snapshot; not visible |

**Decision tree** (simplified from `mvcc.c`):

```
if (no valid delete id):
  if (no valid insert id) → SNAPSHOT_SATISFIED   // vacuumed = all-visible
  if (inserted by me)    → SNAPSHOT_SATISFIED
  if (inserter in snapshot) → TOO_NEW_FOR_SNAPSHOT
  else                   → SNAPSHOT_SATISFIED     // inserter committed before snapshot

if (valid delete id):
  if (deleted by me)     → TOO_OLD_FOR_SNAPSHOT
  if (inserter in snapshot) → TOO_NEW_FOR_SNAPSHOT
  if (deleter in snapshot)  → SNAPSHOT_SATISFIED  // deleter still running, not yet deleted
  else                   → TOO_OLD_FOR_SNAPSHOT   // deleter committed before snapshot
```

> [!key-insight] Three-range ID check
> `mvcc_is_id_in_snapshot()` avoids touching the transaction table for the majority of records: IDs below `lowest_active_mvccid` are committed (not in snapshot); IDs ≥ `highest_completed_mvccid` are active (in snapshot). Only the uncertain middle range consults the `m_active_mvccs` bitset.

### `mvcc_satisfies_delete` — DML path

Used during `DELETE` / `UPDATE` to decide whether the current transaction can modify a record. Returns `MVCC_SATISFIES_DELETE_RESULT`:

| Result | Action |
|--------|--------|
| `DELETE_RECORD_CAN_DELETE` | Row is visible and lockable |
| `DELETE_RECORD_DELETED` | Already committed-deleted; skip |
| `DELETE_RECORD_DELETE_IN_PROGRESS` | Another active transaction deleted it; block and retry |
| `DELETE_RECORD_SELF_DELETED` | This transaction already deleted it |
| `DELETE_RECORD_INSERT_IN_PROGRESS` | Inserter still active; row not yet stable |

### `mvcc_satisfies_vacuum` — Vacuum GC path

Determines whether a record (and its `prev_version_lsa` chain) can be physically reclaimed. Takes `oldest_mvccid` — the lowest MVCCID of any active snapshot in the system.

| Result | Meaning |
|--------|---------|
| `VACUUM_RECORD_REMOVE` | Record entirely dead; can be removed |
| `VACUUM_RECORD_DELETE_INSID_PREV_VER` | Only clear insert MVCCID and prev_version_lsa |
| `VACUUM_RECORD_CANNOT_VACUUM` | Record too recent or already vacuumed |

### `mvcc_satisfies_dirty` — Dirty read / index scan

Used for index scans where rows from own transaction need to be visible even without a committed snapshot.

## MVCC-Disabled Classes

`mvcc_is_mvcc_disabled_class(class_oid)` returns `true` for system catalog tables that bypass MVCC. These tables use conventional locking only — no `MVCC_REC_HEADER` stamping.

## Log Record Integration

MVCC operations produce `LOG_MVCC_UNDOREDO_DATA` / `LOG_MVCC_UNDO_DATA` / `LOG_MVCC_REDO_DATA` log record types (types 46–49). These embed `MVCCID` and `LOG_VACUUM_INFO` (previous MVCC op LSA + VFID) so that:
1. Vacuum can walk the MVCC operation log chain backward.
2. Recovery can re-stamp MVCC headers correctly on redo.

The recovery index macros `LOG_IS_MVCC_HEAP_OPERATION()` and `LOG_IS_MVCC_BTREE_OPERATION()` identify whether a `rcvindex` belongs to the MVCC path.

## Version Chain Navigation

When `mvcc_satisfies_snapshot` returns `TOO_NEW_FOR_SNAPSHOT`, the caller (typically heap scan in [[components/heap-file|heap-file]]) follows `prev_version_lsa` backward through the log to find the version that was current at snapshot time. This is the CUBRID "version chain" — older versions live as undo images in the log, not in the heap.

> [!warning] Version chain in log, not heap
> Unlike PostgreSQL (which stores old versions in heap as dead tuples), CUBRID keeps old versions only in the WAL. This means old-version reads require log I/O. Vacuum clears `prev_version_lsa` when all snapshots are past the point where the old version could be needed.

## Performance Tracking

All visibility decisions instrument `perfmon_mvcc_snapshot()` when `PERFMON_ACTIVATION_FLAG_MVCC_SNAPSHOT` is active. This tracks per-decision-path counters (inserted-vacuumed-visible, deleted-curr-tran-invisible, etc.) accessible via `SHOW SERVER STATUS`.

## Related

- Parent: [[components/transaction|transaction]]
- [[components/heap-file|heap-file]] — embeds `MVCC_REC_HEADER` in every heap record; calls `mvcc_satisfies_snapshot`
- [[components/btree|btree]] — 18 `btree_op_purpose` values map to MVCC operations; uses `mvcc_satisfies_snapshot` during index scans
- [[components/vacuum|vacuum]] — calls `mvcc_satisfies_vacuum`; reclaims dead versions
- [[components/log-manager|log-manager]] — MVCC log records (`LOG_MVCC_*`) written by WAL
- [[components/query-reevaluation|query-reevaluation]] — re-runs predicates on current row version when MVCC detects concurrent modification during scan
- Source: [[sources/cubrid-src-transaction]]
