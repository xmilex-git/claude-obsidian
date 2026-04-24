---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/scan_manager.c"
status: active
purpose: "Unified scan abstraction: open/next/close dispatch for heap, index, list, set, JSON, method, dblink, and parallel scans"
key_files:
  - "scan_manager.c (main implementation)"
  - "scan_manager.h (SCAN_ID, SCAN_TYPE, all scan structs, public API)"
  - "query_hash_scan.c / query_hash_scan.h (hash list scan)"
public_api:
  - "scan_open_heap_scan(...) → int"
  - "scan_open_index_scan(...) → int"
  - "scan_open_list_scan(...) → int"
  - "scan_open_dblink_scan(...) → int"
  - "scan_open_set_scan / scan_open_json_table_scan / scan_open_method_scan / ..."
  - "scan_start_scan(thread_p, scan_id) → int"
  - "scan_next_scan(thread_p, scan_id) → SCAN_CODE"
  - "scan_prev_scan(thread_p, scan_id) → SCAN_CODE"
  - "scan_end_scan(thread_p, scan_id)"
  - "scan_close_scan(thread_p, scan_id)"
  - "scan_save_scan_pos / scan_jump_scan_pos"
  - "scan_print_stats_json / scan_print_stats_text"
tags:
  - component
  - cubrid
  - scan
  - query
related:
  - "[[components/query|query]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/list-file|list-file]]"
  - "[[components/dblink|dblink]]"
  - "[[components/partition-pruning|partition-pruning]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# `scan_manager.c` — Scan Abstraction Layer

Provides a uniform open/next/close interface over every storage-layer scan type CUBRID supports. The query executor calls `scan_next_scan()` without knowing whether it is iterating a heap file, a B-tree, a temp list file, a hash table, a remote dblink query, or a JSON table.

## SCAN_ID: The Polymorphic Scan Handle

```c
struct scan_id_struct {           // SCAN_ID
  SCAN_TYPE    type;              // dispatch key
  SCAN_STATUS  status;
  SCAN_POSITION position;
  SCAN_DIRECTION direction;       // forward / backward
  bool         mvcc_select_lock_needed;
  SCAN_OPERATION_TYPE scan_op_type; // SELECT / DELETE / UPDATE
  int fixed;                      // keep pages fixed across group
  int grouped;                    // group-by mode
  QPROC_SINGLE_FETCH single_fetch;
  QPROC_QUALIFICATION qualification;
  DB_VALUE *join_dbval;           // early-exit join shortcut
  val_list_node *val_list;
  val_descr     *vd;
  union {
    LLIST_SCAN_ID        llsid;   // list file scan
    HEAP_SCAN_ID         hsid;    // heap scan
    PARALLEL_HEAP_SCAN_ID phsid;  // parallel heap scan
    HEAP_PAGE_SCAN_ID    hpsid;   // heap page scan
    INDX_SCAN_ID         isid;    // index (btree) scan
    INDEX_NODE_SCAN_ID   insid;   // b-tree node scan
    SET_SCAN_ID          ssid;    // set scan
    DBLINK_SCAN_ID       dblid;   // dblink remote scan
    REGU_VALUES_SCAN_ID  rvsid;   // VALUES list scan
    SHOWSTMT_SCAN_ID     stsid;   // SHOW statement scan
    JSON_TABLE_SCAN_ID   jtid;    // JSON_TABLE scan
    METHOD_SCAN_ID       msid;    // stored procedure/method scan
  } s;
  SCAN_STATS scan_stats;
  SCAN_STATS *partition_stats;
};
```

## Scan Types

| `SCAN_TYPE` | Description |
|-------------|-------------|
| `S_HEAP_SCAN` | Full heap file scan (row-store) |
| `S_PARALLEL_HEAP_SCAN` | Parallel heap scan (delegates to `parallel/`) |
| `S_CLASS_ATTR_SCAN` | Class-level attribute scan (schema info) |
| `S_INDX_SCAN` | B-tree index scan (range, eq, covering index, ISS, MRO) |
| `S_LIST_SCAN` | Temp list file scan (intermediate results) |
| `S_SET_SCAN` | SET/MULTISET/SEQUENCE element scan |
| `S_JSON_TABLE_SCAN` | JSON_TABLE() virtual table |
| `S_METHOD_SCAN` | Method/stored procedure invocation |
| `S_VALUES_SCAN` | VALUES(...) literal list |
| `S_SHOWSTMT_SCAN` | SHOW TABLES / SHOW INDEX / etc. |
| `S_HEAP_SCAN_RECORD_INFO` | Heap scan with raw record info (checkdb) |
| `S_HEAP_PAGE_SCAN` | Page-level scan for page header info |
| `S_INDX_KEY_INFO_SCAN` | B-tree key metadata scan |
| `S_INDX_NODE_INFO_SCAN` | B-tree node metadata scan |
| `S_DBLINK_SCAN` | Remote CUBRID node query via CCI |
| `S_HEAP_SAMPLING_SCAN` | Statistical sampling scan |

## Index Scan Optimizations

The `INDX_SCAN_ID` embeds three optional optimizations:

| Optimization | Flag / struct | Description |
|---|---|---|
| Covering Index | `INDX_COV indx_cov` | All needed columns in index key — no heap fetch |
| Multi-Range Opt | `MULTI_RANGE_OPT multi_range_opt` | Top-N key results held in memory; range search aborts early |
| Index Skip Scan | `INDEX_SKIP_SCAN iss` | Skip over leading column gaps; avoids full scan for partial-key predicates |
| Loose Index Scan | tracked in `SCAN_STATS.loose_index_scan` | Efficient distinct-value iteration |

> [!key-insight] join_dbval early-exit
> `SCAN_ID.join_dbval` is a single `DB_VALUE*` from a join partner. If it is set but unbound (IS NULL equivalent), `scan_next_scan` returns `S_END` immediately without touching storage. This is a cheap NL-join short-circuit that avoids heap/index lookups for null FK situations.

## `scan_next_scan` Dispatch

```c
SCAN_CODE scan_next_scan(THREAD_ENTRY *thread_p, SCAN_ID *s_id) {
    // calls scan_next_scan_local() which dispatches on s_id->type:
    //   S_HEAP_SCAN         → scan_next_heap_scan()
    //   S_PARALLEL_HEAP_SCAN→ scan_next_parallel_heap_scan()
    //   S_INDX_SCAN         → scan_next_index_scan()
    //   S_LIST_SCAN         → scan_next_list_scan()
    //   S_SET_SCAN          → scan_next_set_scan()
    //   S_JSON_TABLE_SCAN   → scan_next_json_table_scan()
    //   S_METHOD_SCAN       → scan_next_method_scan()
    //   S_DBLINK_SCAN       → dblink_scan_next()
    //   ...
}
```

## Scan Statistics

`SCAN_STATS` tracks: elapsed time, I/O reads, rows read, rows qualified, key reads, key filter pass rate, and hash join build time. Exposed via `scan_print_stats_json` / `scan_print_stats_text` (server mode only, using jansson).

## Grouped / Blocked Scan Mode

When `grouped = 1`, pages are fixed (pinned) across a block of tuples. Up to `QPROC_MAX_GROUPED_SCAN_CNT = 4` grouped scans are supported for join queries. `QPROC_SINGLE_CLASS_GROUPED_SCAN = 0` (disabled for single-class queries).

## MVCC Integration

`mvcc_select_lock_needed` and `scan_op_type` control whether MVCC snapshot locks are acquired at scan time (needed for UPDATE/DELETE, not for plain SELECT in READ COMMITTED).

## Related

- Parent: [[components/query|query]]
- [[components/query-executor|query-executor]] — calls scan_open_* then scan_next_scan in a loop
- [[components/list-file|list-file]] — `S_LIST_SCAN` target
- [[components/dblink|dblink]] — `S_DBLINK_SCAN` backend
- [[components/parallel-query|parallel-query]] — `S_PARALLEL_HEAP_SCAN` backend
- [[components/partition-pruning|partition-pruning]] — partitioned class access specs pruned before scan open
- [[components/method-scan|method-scan]] — `S_METHOD_SCAN` backend (`METHOD_SCAN_ID msid`)
- [[components/method|method]] — method invocation layer owning `cubscan::method::scanner`
- [[components/query-reevaluation|query-reevaluation]] — MVCC predicate re-evaluation when a scanned row is modified by a concurrent transaction
