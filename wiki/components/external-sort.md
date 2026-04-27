---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/external_sort.c"
status: active
purpose: "External merge sort over temporary list files; single entry point sort_listfile() consumed by both the sequential query executor and by the parallel sort subsystem"
key_files:
  - "external_sort.c — implementation"
  - "external_sort.h — public API, SORT_INFO, SORT_REC, SORTKEY_INFO, SUBKEY_INFO structs"
public_api:
  - "sort_listfile(thread_p, volid, est_inp_pg_cnt, get_fn, get_arg, put_fn, put_arg, cmp_fn, cmp_arg, option, limit, includes_tde_class, sort_parallel_type)"
tags:
  - component
  - cubrid
  - storage
  - sort
  - external-sort
related:
  - "[[components/storage|storage]]"
  - "[[components/parallel-sort|parallel-sort]]"
  - "[[components/file-manager|file-manager]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# `external_sort.c` — External Sort

The external sort subsystem implements a replacement-selection / merge-sort over temporary files. It is the sort primitive used by ORDER BY, GROUP BY, analytic functions, and index leaf loading.

## Entry Point

```c
int sort_listfile (
  THREAD_ENTRY  *thread_p,
  INT16          volid,            /* temp volume hint */
  int            est_inp_pg_cnt,   /* input page count estimate */
  SORT_GET_FUNC *get_fn,           /* callback: fetch next record from input */
  void          *get_arg,
  SORT_PUT_FUNC *put_fn,           /* callback: deliver sorted record to output */
  void          *put_arg,
  SORT_CMP_FUNC *cmp_fn,           /* comparison function */
  void          *cmp_arg,
  SORT_DUP_OPTION option,          /* SORT_ELIM_DUP or SORT_DUP */
  int            limit,            /* top-N limit, or NO_SORT_LIMIT */
  bool           includes_tde_class,
  SORT_PARALLEL_TYPE sort_parallel_type
);
```

Callers provide three callbacks:
- `get_fn`: pulls the next input record (returns `SORT_SUCCESS`, `SORT_NOMORE_RECS`, or `SORT_ERROR_OCCURRED`).
- `put_fn`: receives each sorted output record.
- `cmp_fn`: compares two sort keys.

## Key Data Structures

### `SORT_REC`

```c
struct SORT_REC {
  SORT_REC *next;    /* chained list for duplicate sort keys */
  union {
    struct { INT32 pageid; INT16 volid; INT16 offset; char body[1]; } original;
    int offset[1];   /* column offset vector */
  } s;
};
```

`SORT_RECORD_LENGTH(item_p)` reads the 8-byte length prefix before each sort record.

### `SORTKEY_INFO`

Holds per-query sort key configuration:
- `nkeys` — number of sort columns.
- `key[i]` — `SUBKEY_INFO` per column: column index, domain, comparison function, `is_desc`, `is_nulls_first`.
- Default storage for up to 8 keys (`default_keys[8]`); larger sets malloc additional space.

### `SORT_INFO`

Top-level context:
- Input list file scan state (`s_id`), output list file (`output_file`).
- `parallelism` — set by the parallel sort subsystem.
- `sort_list_p`, `flag` — passed to the output list file opener.

## Duplicate Handling

`SORT_DUP_OPTION`:
- `SORT_ELIM_DUP` — identical keys produce one output record (useful for `DISTINCT`).
- `SORT_DUP` — all records, including duplicates, are emitted.

## Parallel Sort Integration

> [!key-insight] sort_listfile is the bridge to parallel sort
> When `sort_parallel_type` is set and `parallelism > 1`, `sort_listfile` internally invokes `SORT_EXECUTE_PARALLEL` / `SORT_WAIT_PARALLEL` macros defined in [[components/parallel-sort|px_sort.h]]. The parallel sort subsystem splits input runs across worker threads, then merges the sorted runs on the main thread.

`SORT_PARALLEL_TYPE` enum:

| Value | Context |
|-------|---------|
| `SORT_ORDER_BY` | ORDER BY clause |
| `SORT_ORDER_WITH_LIMIT` | ORDER BY … LIMIT (allows early termination) |
| `SORT_GROUP_BY` | GROUP BY |
| `SORT_ANALYTIC` | Window functions |
| `SORT_INDEX_LEAF` | B-tree leaf sort during index load (fully wired in `cc563c7` / PR #7011 — see "Index-leaf parallel build" below) |

## Index-leaf parallel build (`SORT_INDEX_LEAF`)

> [!update] PR #7011 (merge `cc563c7f`) — `SORT_INDEX_LEAF` fully wired
> The enum value existed (declared since PR #5694) but was never dispatched. PR #7011 adds `SORT_INDEX_LEAF` arms to `sort_listfile`, `sort_listfile_execute`, `sort_return_used_resources`, `sort_check_parallelism`, `sort_start_parallelism`, and `sort_end_parallelism`, plus the parallel tree-merge `sort_merge_run_for_parallel_index_leaf_build`.

`sort_merge_run_for_parallel_index_leaf_build` (defined at `external_sort.c:4733`) implements a **log₄ tree-merge** over per-worker temp files: `SORT_PX_MERGE_FILES = 4` is the fan-in degree; depth = `ceil(log₄ parallel_num)` (e.g. 8 → 2 levels, 64 → 3 levels). Empty (0-page) workers are skipped during input gathering — only non-empty runs are passed into a merge slot. If `valid_j ≤ 1` for a slot, no merge is dispatched (the lone non-empty run propagates directly). After all merges complete, parent `SORT_ARGS` accumulates `n_oids` and `n_nulls` from per-worker copies. The function also performs the final `btree_create_file` + `vacuum_log_add_dropped_file` + `log_sysop_start` for the SERVER_MODE parallel path (lines 4848-4858), then `sort_put_result_from_tmpfile` to deliver the sorted output.

`sort_listfile` SERVER_MODE single-process arm wraps scancache (`bt_load_heap_scancache_start_for_attrinfo`) + `btree_create_file` + `log_sysop_start` around `sort_listfile_internal`, and `bt_load_heap_scancache_end_for_attrinfo` after it returns (`external_sort.c:1517`).

> [!info] SA_MODE / SERVER_MODE single-process asymmetry
> `sort_listfile`'s SA_MODE branch (`external_sort.c:1559-1561`) skips the scancache + sysop + `btree_create_file` setup that the SERVER_MODE single-process branch performs — `btree_load.c::xbtree_load_index` keeps its own setup under `#if !defined(SERVER_MODE)` (`btree_load.c:1078-1085`). Ownership of scancache lifecycle differs by mode: a future SA_MODE consumer of `sort_listfile(..., SORT_INDEX_LEAF)` outside `xbtree_load_index` would silently miss the setup.

PR #7011 also lands two silent fixes the body did not mention:
- `external_sort.c:1491` — `if (sort_param == NULL)` → `if (px_sort_param == NULL)` after the worker-array `malloc` (was always-true on the wrong pointer).
- `external_sort.c:1102` — `malloc(...)` → `calloc(...)` for `file_contents[j].num_pages`. Avoids a possible uninit read for empty-worker runs in the tree merge.

## Top-N Optimization

When `limit > 0` (not `NO_SORT_LIMIT`), the sort can use a bounded heap to keep only the top-N records in memory, avoiding a full external merge when the limit is small relative to input size.

## TDE Support

`includes_tde_class` causes temp file pages to be created with TDE encryption. This is required when the sort input contains data from TDE-encrypted tables.

## Related

- Parent: [[components/storage|storage]]
- [[components/parallel-sort]] — consumes `sort_listfile` with `parallelism > 1`
- [[components/file-manager]] — temp files allocated from `FILE_TEMP` / `FILE_QUERY_AREA`
- [[Query Processing Pipeline]] — called by `qexec_execute_mainblock` for ORDER BY / GROUP BY
