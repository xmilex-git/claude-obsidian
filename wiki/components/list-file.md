---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/list_file.c"
status: active
purpose: "Intermediate and final query result spooling in temp list files; sorting, set operations, and query result cache"
key_files:
  - "list_file.c (qfile_* implementation)"
  - "list_file.h (QFILE_LIST_CACHE_ENTRY, QFILE_PAGE_HEADER, public API)"
  - "query_list.h (QFILE_LIST_ID, QFILE_TUPLE*, scan types — shared)"
  - "external_sort.h / external_sort.c (sort backend used by qfile_sort_list)"
public_api:
  - "qfile_open_list(thread_p, type_list, sort_list, query_id, flag, ...) → QFILE_LIST_ID*"
  - "qfile_add_tuple_to_list(thread_p, list_id, tpl) → int"
  - "qfile_sort_list(thread_p, list_id, sort_list, option, do_close) → QFILE_LIST_ID*"
  - "qfile_sort_list_with_func(..., parallelism) → QFILE_LIST_ID*"
  - "qfile_combine_two_list(thread_p, lhs, rhs, flag) → QFILE_LIST_ID*"
  - "qfile_close_list / qfile_destroy_list / qfile_free_list_id"
  - "qfile_open_list_scan / qfile_scan_list_next / qfile_scan_list_prev / qfile_close_scan"
  - "qfile_lookup_list_cache_entry / qfile_update_list_cache_entry"
  - "qfile_clear_list_cache / qfile_end_use_of_list_cache_entry"
  - "qfile_fast_intint_tuple_to_list / qfile_fast_intval_tuple_to_list / qfile_fast_val_tuple_to_list"
  - "qfile_collect_list_sector_info / qfile_free_list_sector_info (sector-based parallel scan helpers; PR #6981)"
tags:
  - component
  - cubrid
  - query
  - list-file
  - sort
  - result-cache
related:
  - "[[components/query|query]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/cursor|cursor]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# `list_file.c` — Temp Result Spool

List files are CUBRID's mechanism for materializing intermediate and final query results. They are server-side temp volumes (allocated from the temp file subsystem) organized as linked pages of fixed-format tuples. Every `BUILDLIST_PROC` node in the XASL tree writes its output to a list file; set operations and sort passes consume and produce list files.

## QFILE_LIST_ID

```c
// defined in query_list.h (shared client+server)
typedef struct qfile_list_id QFILE_LIST_ID;
```

Key fields: page-count, type list (`QFILE_TUPLE_VALUE_TYPE_LIST`), sort list, tuple count, `QUERY_ID`, backing `QMGR_TEMP_FILE *tfile_vfid` (NULL if the list is purely in memory). `QFILE_LIST_ID` is what `qexec_execute_query` returns to the client; the cursor wraps it.

### Dependent-list chain

A `QFILE_LIST_ID` may reference another via the `dependent_list_id` field (singly-linked). This appears when one intermediate result feeds another — e.g. the inner side of a nested-loop join whose rows are materialised into a secondary list. Consumers iterating the primary list must also iterate every dependent in the chain to produce the full tuple stream; the parallel list scan's input handler walks the chain and aggregates all sectors from base plus dependents into a single `FILE_FTAB_COLLECTOR` before splitting work across workers.

### Membuf vs disk backing

`QMGR_TEMP_FILE` has two storage backends that co-exist:

- **Membuf** — a short fixed-size page array held in memory. Populated first; small lists (typical intermediate results from selective queries) never leave membuf. Membuf pages carry `volid = NULL_VOLID`.
- **Temp file** — disk-resident, allocated via the file manager when membuf fills up. `temp_vfid` identifies the file; data-bearing sectors are harvested via `file_get_all_data_sectors` (see [[components/file-manager]]).

A list can be (a) membuf-only, (b) membuf + disk (membuf pages are still consumed first), or (c) disk-only after spill. The parallel list scan dedicates worker 0 to any membuf pages present (Phase 1), then all workers iterate disk sectors in parallel (Phase 2).

### `QFILE_OVERFLOW_TUPLE_COUNT_FLAG`

Wide tuples that exceed a page's capacity spill to an overflow chain. Overflow pages carry the sentinel tuple-count marker `QFILE_OVERFLOW_TUPLE_COUNT_FLAG = -2` in their page header. Sector-level parallel scans must detect this marker after a speculative page read and skip the page — overflow pages are already consumed by the primary page's `qfile_get_tuple` handler, so processing them again would double-count.

## Page Layout

```c
struct qfile_page_header {
  int    pg_tplcnt;    // tuple count in this page
  PAGEID prev_pgid;    // doubly-linked list
  PAGEID next_pgid;
  int    lasttpl_off;  // offset of last tuple (for append)
  PAGEID ovfl_pgid;    // overflow chain for wide tuples
  VOLID  prev_volid;
  VOLID  next_volid;
  VOLID  ovfl_volid;
};
```

Pages are chained forward and backward. Wide tuples that exceed a page size spill to an overflow chain.

> [!key-insight] List file I/O is the primary bottleneck for large result sets
> When intermediate list files exceed the temp file memory buffer, they hit disk. Query plans that minimize list-file size (covering indexes, early predicate pushdown, hash aggregate in-memory path) have disproportionate impact on performance.

## Tuple Insertion

```
qfile_generate_tuple_into_list()   ← general path
qfile_fast_intint_tuple_to_list()  ← fast path: two integers
qfile_fast_intval_tuple_to_list()  ← fast path: int + DB_VALUE
qfile_fast_val_tuple_to_list()     ← fast path: single DB_VALUE
```

Fast-path functions bypass the full `QFILE_TUPLE_DESCRIPTOR` building for hot aggregation loops.

## Sorting

```c
QFILE_LIST_ID *qfile_sort_list(thread_p, list_id, sort_list, option, do_close);
QFILE_LIST_ID *qfile_sort_list_with_func(thread_p, ..., parallelism, ...);
```

Delegates to `external_sort.c` (external merge sort). `qfile_sort_list_with_func` accepts a `parallelism` parameter wired to [[components/parallel-query|parallel-sort]].

Sort key construction uses `qfile_make_sort_key` / `qfile_generate_sort_tuple` / `qfile_compare_partial_sort_record`.

## Set Operations

```c
QFILE_LIST_ID *qfile_combine_two_list(thread_p, lhs, rhs, flag);
```

`flag` selects UNION, INTERSECT, or EXCEPT semantics. Both inputs must be sorted.

## Query Result Cache

List files can be cached across transactions for identical queries (parameterized cache keyed by `XASL_CACHE_ENTRY + DB_VALUE_ARRAY`):

```c
struct qfile_list_cache_entry {
  int list_ht_no;                // hash table slot
  DB_VALUE_ARRAY param_values;   // bound parameter snapshot
  QFILE_LIST_ID list_id;         // the cached result
  QFILE_LIST_CACHE_ENTRY *tran_next;  // transaction list chain
  TRAN_ISOLATION tran_isolation;
  bool uncommitted_marker;       // producing txn not yet committed
  int *tran_index_array;         // transactions currently reading this entry
  XASL_CACHE_ENTRY *xcache_entry;
  struct timeval time_created;
  struct timeval time_last_used;
  int ref_count;
  bool deletion_marker;
  bool invalidate;
};
```

Cache modes: `OFF`, `SELECTIVELY_OFF`, `SELECTIVELY_ON` (controlled by `QFILE_IS_LIST_CACHE_DISABLED`).

> [!warning] Isolation-aware caching
> A cached list entry records the producing transaction's isolation level (`tran_isolation`) and an `uncommitted_marker`. Readers at stricter isolation levels must not see entries produced by lower-isolation or uncommitted transactions. The cache entry reference count (`tran_index_array`) tracks concurrent readers.

## Scan API

```
qfile_open_list_scan(list_id, s_id)
qfile_scan_list_next(thread_p, s_id, tplrec, peek)  → SCAN_CODE
qfile_scan_list_prev(thread_p, s_id, tplrec, peek)  → SCAN_CODE
qfile_close_scan(thread_p, s_id)
```

Used by both the `S_LIST_SCAN` path in `scan_manager.c` and directly by aggregate / analytic functions that re-scan their temp list.

## Related

- Parent: [[components/query|query]]
- [[components/query-executor|query-executor]] — produces and consumes list files
- [[components/scan-manager|scan-manager]] — `S_LIST_SCAN` uses `qfile_open_list_scan`
- [[components/cursor|cursor]] — client reads the final list file via `CURSOR_ID`
- [[components/aggregate-analytic|aggregate-analytic]] — hash aggregate spills to list file; analytic sort uses list file
