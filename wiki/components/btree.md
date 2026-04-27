---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/btree.c"
status: active
purpose: "B+tree index manager: point lookup, range scan, insert, MVCC delete, bulk load, unique constraint tracking, and online index build"
key_files:
  - "btree.c (~37K lines) — full implementation; use function index to navigate"
  - "btree.h — public API, BTREE_SCAN, BTID_INT, BTREE_OP_PURPOSE enums"
  - "btree_load.c / btree_load.h — bulk index loading (sorted insert), page constants"
  - "btree_unique.hpp — btree_unique_stats / multi_index_unique_stats (C++ classes)"
public_api:
  - "btree_insert(thread_p, btid, key, cls_oid, oid, op_type, unique_stat_info, unique, mvcc_header)"
  - "btree_mvcc_delete(thread_p, btid, key, class_oid, oid, op_type, ...)"
  - "btree_update(thread_p, btid, old_key, new_key, cls_oid, oid, op_type, ...)"
  - "btree_keyval_search(thread_p, btid, scan_op_type, bts, key_val_range, ...)"
  - "btree_range_scan(thread_p, bts, key_func)"
  - "btree_range_scan_select_visible_oids(thread_p, bts)"
  - "btree_prepare_bts(thread_p, bts, btid, index_scan_id_p, key_val_range, ...)"
  - "btree_find_key(thread_p, btid, oid, key, clear_key)"
  - "btree_find_min_or_max_key(thread_p, btid, key, flag_minkey)"
  - "btree_locate_key(thread_p, btid_int, key, pg_vpid, slot_id, leaf_page_out, found_p)"
  - "btree_online_index_dispatcher / btree_online_index_list_dispatcher"
  - "btree_vacuum_object / btree_vacuum_insert_mvccid"
tags:
  - component
  - cubrid
  - storage
  - btree
  - index
related:
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/transaction|transaction]]"
created: 2026-04-23
updated: 2026-04-23
---

# `btree.c` — B+Tree Index Manager

CUBRID's index engine implements a B+tree (leaves linked left-to-right) with full MVCC support, online index builds, and key prefix compression. The main file is ~37 K lines; navigating it requires a function index approach.

> [!warning] btree.c size
> At ~37 K lines, `btree.c` is one of the largest files in the engine. Do not attempt to read it top-to-bottom. Use the function table below to jump to relevant sections.

## Node Structure

| Node type | Enum | Description |
|-----------|------|-------------|
| `BTREE_LEAF_NODE` | 0 | Leaf: stores `(key, OID+MVCC_INFO)` tuples, linked horizontally |
| `BTREE_NON_LEAF_NODE` | 1 | Internal: stores `(key, child_VPID)` pairs |
| `BTREE_OVERFLOW_NODE` | 2 | Overflow OID pages for keys with many duplicates |

### Key Internal Structs

```c
/* Per-record header in a leaf or non-leaf slot */
struct leaf_rec {
  VPID ovfl;        /* first overflow OID page, or NULL */
  short key_len;
};
struct non_leaf_rec {
  VPID pnt;         /* child page pointer */
  short key_len;
};

/* Full B-tree descriptor (passed through all internal calls) */
struct btid_int {
  BTID        *sys_btid;
  int          unique_pk;          /* is unique / is primary key flags */
  int          part_key_desc;      /* last partial-key domain is DESC */
  TP_DOMAIN   *key_type;
  TP_DOMAIN   *nonleaf_key_type;   /* may differ from key_type for prefix keys */
  VFID         ovfid;              /* overflow key file */
  int          rev_level;
  OID          topclass_oid;
};
```

### MVCC Info in Leaf Records

```c
struct btree_mvcc_info {
  short    flags;           /* HAS_INSID | HAS_DELID flags */
  MVCCID   insert_mvccid;
  MVCCID   delete_mvccid;
};
```

Every live row in a leaf record carries its insert MVCCID; deleted rows additionally carry a delete MVCCID. Vacuum (`btree_vacuum_insert_mvccid`, `btree_vacuum_object`) removes these once the MVCCID becomes globally visible.

## Scan Architecture (`BTREE_SCAN`)

`BTREE_SCAN` (BTS) is the per-scan cursor. Key fields:

| Field | Purpose |
|-------|---------|
| `btid_int` | B-tree descriptor |
| `C_vpid`, `C_page` | Current leaf page |
| `P_vpid`, `P_page` | Previous leaf page (for backward scan) |
| `slot_id` | Current slot within leaf |
| `cur_key` | Current key value (`DB_VALUE`) |
| `key_range` | Lower/upper bounds (`BTREE_KEYRANGE`) |
| `key_filter` | Post-key predicate filter |
| `use_desc_index` | Descending scan flag |
| `common_prefix_size` / `common_prefix_key` | Key compression state |
| `lock_mode` | S_LOCK or X_LOCK (for next-key locking) |
| `end_scan` | Scan exhausted |

Initialize with `BTREE_INIT_SCAN(bts)` macro; reset between iterations with `BTREE_RESET_SCAN`.

## Range Scan Flow

```
btree_prepare_bts()           ← set up BTS with key range + filter
        │
        ▼
btree_range_scan()            ← iterate leaf pages
        │
        │  for each qualifying key:
        ▼
BTREE_RANGE_SCAN_PROCESS_KEY_FUNC  (e.g. btree_range_scan_select_visible_oids)
        │
        ▼
MVCC snapshot check           ← btree_range_scan_select_visible_oids
        │
        ▼
OID list returned to scan_manager
```

`btree_range_scan_select_visible_oids` applies the MVCC snapshot to each `(OID, insert_mvccid, delete_mvccid)` tuple, accepting only rows visible to the current transaction.

## Key Compression

> [!key-insight] Common-prefix key compression
> Leaf pages store a `common_prefix_key` per page. Individual keys store only the suffix that differs. `is_cur_key_compressed` in `BTREE_SCAN` signals that `cur_key` must be concatenated with `common_prefix_key` before use. The compression state is validated in debug builds via `CHECK_VERIFY_COMMON_PREFIX_PAGE_INFO`.

## MVCC Operation Purposes (`btree_op_purpose`)

B-tree insert/delete paths are parameterized by `BTREE_OP_PURPOSE` to handle all MVCC and recovery combinations:

| Purpose | Meaning |
|---------|---------|
| `BTREE_OP_INSERT_NEW_OBJECT` | Normal insert with insert MVCCID |
| `BTREE_OP_INSERT_MVCC_DELID` | Mark existing row as deleted (MVCC delete) |
| `BTREE_OP_INSERT_MARK_DELETED` | Non-MVCC class mark-delete (e.g. `_db_serial`) |
| `BTREE_OP_DELETE_OBJECT_PHYSICAL` | Physically remove entry (after vacuum or commit) |
| `BTREE_OP_DELETE_UNDO_INSERT` | Undo of an insert |
| `BTREE_OP_DELETE_VACUUM_OBJECT` | Vacuum removes fully invisible entry |
| `BTREE_OP_DELETE_VACUUM_INSID` | Vacuum removes insert MVCCID only |
| `BTREE_OP_ONLINE_INDEX_IB_INSERT` | Online index builder insert |
| `BTREE_OP_ONLINE_INDEX_TRAN_INSERT` | Concurrent transaction insert during index build |

## Unique Constraint Tracking (`btree_unique.hpp`)

```cpp
class btree_unique_stats {
  stat_type m_rows;    // rows = keys + nulls
  stat_type m_keys;    // distinct non-null keys
  stat_type m_nulls;   // null key rows
  bool is_unique() const { return m_rows == m_keys + m_nulls; }
};

class multi_index_unique_stats {
  std::map<BTID, btree_unique_stats, btid_comparator> m_stats_map;
};
```

`multi_index_unique_stats` is embedded in `HEAP_SCANCACHE` to accumulate per-index stats during a DML operation. On commit, `btree_reflect_global_unique_statistics` merges local counts into the global stats on the index root page.

> [!key-insight] Unique check timing
> Unique violations are checked at the end of the statement (or transaction for deferred constraints), not inline during each insert. `BTREE_NEED_UNIQUE_CHECK` returns `true` for single/multi-row insert and single-row update, but only when the transaction is active.

## Bulk Loading (`btree_load.c`)

Bulk index creation (e.g. `CREATE INDEX`) uses a sorted-insert path optimized for sequential page writes:
- Fill factor controlled by `PRM_ID_BT_UNFILL_FACTOR` (applied to both leaf and non-leaf via `LOAD_FIXED_EMPTY_FOR_LEAF` / `LOAD_FIXED_EMPTY_FOR_NONLEAF`).
- `btree_insert_list` struct (in `btree.h`) batches key-OID pairs and can use `m_use_sorted_bulk_insert` for page-boundary-aware insertion.
- `page_key_boundary` tracks the min/max key of each page to decide when to release the current page latch.

## Latch Ordering

> [!warning] Parent-before-child latch order
> When traversing the tree (insert, delete, split, merge), latches must be acquired parent-before-child. Violating this order causes deadlocks. The tree uses a crabbing protocol: the parent page is released only after the child page is fixed and determined to be safe (no split/merge needed).

## Recovery Functions

Every structural change has a corresponding `btree_rv_*` function pair (`undo` / `redo`):

- `btree_rv_keyval_undo_insert` / `btree_rv_keyval_undo_delete`
- `btree_rv_redo_record_modify` / `btree_rv_undo_record_modify`
- `btree_rv_nodehdr_undoredo_update`, `btree_rv_newpage_redo_init`
- `btree_rv_redo_global_unique_stats_commit` / `btree_rv_undo_global_unique_stats_commit`

## Fence Keys

Leaf pages use fence keys (OID-marker sentinels) to bound the key range of each page. `btree_capacity.fence_key_cnt` counts them; they are invisible to query results but critical for correct range scans and merges.

`btree_leaf_record_is_fence(RECDES *)` is the public helper (`btree.h`) that classifies a slot as a fence. Callers that iterate leaf slots directly — e.g. parallel index scans that bypass the single-thread cursor — **must call this before treating a record as a key**, otherwise boundary entries duplicated across adjacent leaves cause double-counting in aggregates and GROUP BY. The check is only valid after `spage_get_record` succeeds; the fence record's OID slot carries the sentinel marker rather than a real OID.

## Leaf-page header and leaf-chain pointers

`BTREE_NODE_HEADER` is stored as record 0 of every B-tree page (leaf or internal). For leaf pages the header carries:

- `next_vpid` — the VPID of the next leaf in ascending-key order; `VPID_ISNULL` marks the right end of the leaf chain.
- `prev_vpid` — the VPID of the previous leaf; `VPID_ISNULL` marks the left end.
- `node_level` — `1` for leaf, `>1` for internal nodes.
- `max_key_len`, `num_oids`, `num_nulls`, `num_keys` — cardinality metadata.

The forward/backward leaf chain is what makes `btree_range_scan` and parallel leaf-cursor iteration possible without repeated root traversals. The cursor in parallel index scan (`src/query/parallel/px_scan/px_scan_input_handler_index.cpp`) advances by reading `next_vpid` (or `prev_vpid` under `use_desc_index`) inside a mutex and then releasing it.

## Non-leaf record byte layout

Non-leaf records carry `(child_vpid, first_key)` pairs. The first 6 bytes of the record's `data` buffer are **raw-parsable** as the child VPID:

| Offset | Size | Field | Parse macro |
|---|---|---|---|
| `0` | 4 bytes | `pageid` (INT32) | `OR_GET_INT(rec.data)` |
| `4` | 2 bytes | `volid` (INT16) | `OR_GET_SHORT(rec.data + OR_INT_SIZE)` |
| `6` | variable | key data | `btree_read_fixed_portion_of_non_leaf_record` |

This byte layout is a **hidden contract** — callers that descend root-to-leaf without going through `btree_find_first()` (such as `parallel_scan::input_handler_index::init_on_main`) parse these offsets directly. Any future change to the non-leaf record format must keep this prefix compatible or update every direct parser in sync.

## MVCC visibility filtering during B-tree iteration

`btree_mvcc_info_to_heap_mvcc_header(BTREE_MVCC_INFO *, MVCC_REC_HEADER *)` converts the per-OID MVCC fields stored in a leaf record into the heap-header form that `mvcc_satisfies_snapshot` expects. Used by object-processing callbacks — e.g. the parallel-index `collect_oid_callback` in `px_scan_slot_iterator_index.cpp` — to filter out OIDs that are invisible to the current snapshot before returning them to the executor. This is why filtered indexes and mid-scan UPDATE/DELETE do not produce stale results from the btree path.

`btree_key_process_objects(thread_p, btid, key, leaf_rec, BTREE_PROCESS_OBJECT_FUNCTION *fn, void *args)` is the public iteration API over all OIDs (base + overflow chain) for a given leaf key, invoking `fn` per OID; the callback decides whether to accept, reject, or stop. `BTREE_PROCESS_OBJECT_FUNCTION` is the typedef for the callback signature. Both moved from file-static to `btree.h` public surface for use by parallel-scan machinery.

## Key Functions Summary

| Function | Role |
|----------|------|
| `btree_locate_key` | Binary search on a leaf page for a key |
| `btree_range_scan` | Full range scan iteration loop |
| `btree_keyval_search` | Point lookup (wraps range scan with equality range) |
| `btree_insert` | Insert new `(key, OID)` pair |
| `btree_mvcc_delete` | Add delete MVCCID to an existing entry |
| `btree_physical_delete` | Remove an entry from the tree |
| `btree_vacuum_object` | Vacuum: remove fully invisible entry |
| `btree_online_index_dispatcher` | Dispatch online index op for a single key |
| `btree_online_index_list_dispatcher` | Dispatch online index op for a list of keys |
| `btree_check_foreign_key` | FK constraint check (point lookup) |

## From the Manual (sql/schema/index_stmt.rst, sql/schema/table_stmt.rst — added 2026-04-27)

> [!gap] Documented behavior
> - **Online index build** (`CREATE INDEX ... ONLINE PARALLEL N`, N=1..16) runs in **three locked stages**:
>   1. **SCH_M_LOCK** — add new (empty) index entry to `_db_index`. Invisible to other transactions due to MVCC snapshot.
>   2. **IX_LOCK** — populate the index in **16 MB batches** by scanning the heap.
>   3. **SCH_M_LOCK** — promote: make the index visible.
> - **`WITH ONLINE` is IGNORED under standalone mode** — always single-threaded in SA. (`sql/schema/index_stmt.rst:50-80`).
> - **DEDUPLICATE level (0-14, since CUBRID 11.3)** — leaf-page key compression. Level **0 = pre-11.2 layout** (no dedup). Per-table default + per-index override via `DEDUPLICATE` clause. (`sql/schema/table_stmt.rst:90-120`, `sql/schema/index_stmt.rst:30-50`).
> - **Filtered indexes** (`CREATE INDEX ... WHERE <pred>`), **function-based indexes** (`CREATE INDEX ... ON tbl(UPPER(col))`), and **INVISIBLE** indexes (created but ignored by optimizer) are first-class.
> - **Length limit on filter-index WHERE removed in 11.4** (was capped before).
> - **midxkey.buf size optimized in 11.4** for multi-column indexes — direct OFFSET reference, no recalculation. Improves binary search, key filtering, DML.

See [[sources/cubrid-manual-sql-ddl]] for the full DDL reference.

## Related

- Parent: [[components/storage|storage]]
- [[components/page-buffer]] — all page access via pgbuf_fix/unfix
- [[components/heap-file]] — provides OIDs; heap attributes drive key generation
- [[components/transaction]] — lock manager for next-key locking; MVCC for visibility
- Manual: [[sources/cubrid-manual-sql-ddl]]
