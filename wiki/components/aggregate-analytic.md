---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/"
status: active
purpose: "Aggregate function evaluation (GROUP BY / HAVING) and window/analytic function execution (OVER clause)"
key_files:
  - "query_aggregate.hpp (AGGREGATE_HASH_CONTEXT, aggregate evaluation API)"
  - "query_analytic.hpp (analytic function interface)"
  - "query_analytic.cpp (window function implementation)"
  - "query_opfunc.c (qdata_* aggregate ops: SUM, AVG, COUNT, etc.)"
  - "xasl/xasl_aggregate.hpp (aggregate_list_node, aggregate_accumulator)"
  - "xasl/xasl_analytic.hpp (analytic_list_node)"
public_api:
  - "qdata_initialize_aggregate_list(thread_p, agg_list, query_id) → int"
  - "qdata_evaluate_aggregate_list(thread_p, agg_list, vd, alt_acc_list, use_desc) → int"
  - "qdata_evaluate_aggregate_optimize(thread_p, agg_ptr, hfid, partition_cls_oid) → int"
  - "qdata_evaluate_aggregate_hierarchy(thread_p, agg_ptr, root_hfid, root_btid, helper) → int"
  - "qdata_finalize_aggregate_list(thread_p, agg_list, keep_list_file, sampling) → int"
  - "qdata_initialize_analytic_func(thread_p, func_p, query_id) → int"
  - "qdata_evaluate_analytic_func(thread_p, func_p, vd) → int"
  - "qdata_finalize_analytic_func(thread_p, func_p, is_same_group) → int"
  - "qdata_aggregate_accumulator_to_accumulator(...)"
tags:
  - component
  - cubrid
  - query
  - aggregate
  - analytic
  - window-function
related:
  - "[[components/query|query]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/list-file|list-file]]"
  - "[[components/partition-pruning|partition-pruning]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# Aggregate & Analytic Functions

Two related subsystems handle aggregation in CUBRID: the aggregate evaluation path (GROUP BY with `query_aggregate.hpp` + `query_opfunc.c`) and the window/analytic function path (OVER clause with `query_analytic.cpp`).

## Aggregate Functions

### Execution Model

Aggregates are stored as a linked list of `cubxasl::aggregate_list_node` on the XASL node. Each node holds an `aggregate_accumulator` (running state: `value`, `value2` for two-phase ops, `curr_cnt`).

Three evaluation strategies are selected at runtime:

#### 1. Sort-Based GROUP BY (default)
```
1. Materialize matching rows to list file
2. qfile_sort_list() — sort by group keys
3. qdata_initialize_aggregate_list()
4. Scan sorted list: for each new group key,
     qdata_finalize_aggregate_list() → emit group result
     qdata_initialize_aggregate_list() → reset accumulators
5. Final: qdata_finalize_aggregate_list()
```

#### 2. Hash GROUP BY (in-memory)
```
AGGREGATE_HASH_CONTEXT:
  hash_table (mht_table): group-key → aggregate_hash_value
  state: INIT → BUILD → PROBE → SCAN_PARTIAL → DONE

For each tuple:
  1. Compute group key → aggregate_hash_key
  2. mht_get() hit → qdata_evaluate_aggregate_list() on cached value
  3. Miss → allocate new aggregate_hash_value, insert

Spill path (hash table full):
  1. qdata_save_agg_htable_to_list() → partial list file
  2. qfile_sort_list() → sorted partial list
  3. Merge sorted partials back with qdata_load_agg_hentry_from_list()
```

> [!key-insight] Hash aggregate spill threshold
> Spill is triggered when estimated selectivity exceeds `HASH_AGGREGATE_VH_SELECTIVITY_THRESHOLD = 0.5` after `HASH_AGGREGATE_VH_SELECTIVITY_TUPLE_THRESHOLD = 2000` tuples. Below this threshold CUBRID assumes high group cardinality (each key is unique) and uses the sort-based path anyway. The hash path is most beneficial for low-cardinality grouping.

#### 3. Aggregate Optimization (index-only MIN/MAX)
```
qdata_evaluate_aggregate_optimize()
  → for MIN(pk) or MAX(pk) with no WHERE clause:
     read single leaf from B-tree → O(1) result
     
qdata_evaluate_aggregate_hierarchy()
  → for partitioned tables: apply per-partition, then reduce
```

### Key Structures

```cpp
// namespace cubquery (query_aggregate.hpp)
struct aggregate_hash_key {
  int val_count;
  bool free_values;
  db_value **values;
};
struct aggregate_hash_value {
  int curr_size;
  int tuple_count;
  int func_count;
  aggregate_accumulator *accumulators;
  qfile_tuple_record first_tuple;
};
struct aggregate_hash_context {
  mht_table *hash_table;
  AGGREGATE_HASH_STATE state;
  qfile_list_id *part_list_id;       // partial result spill
  qfile_list_id *sorted_part_list_id;
  SORTKEY_INFO sort_key;
  ...
};
```

### Sampling Support
`qdata_finalize_aggregate_list` accepts a `sampling_info *` for statistical sampling (ANALYZE TABLE path).

## Analytic (Window) Functions

### Interface (`query_analytic.hpp`)

```cpp
int qdata_initialize_analytic_func(thread_p, analytic_list_node*, query_id);
int qdata_evaluate_analytic_func(thread_p, analytic_list_node*, vd);
int qdata_finalize_analytic_func(thread_p, analytic_list_node*, is_same_group);
```

### Execution Model

Window functions require two passes over a partition:
1. **Sort pass** — `qfile_sort_list()` on the PARTITION BY + ORDER BY keys.
2. **Evaluation pass** — scan sorted list; maintain window frame boundaries; evaluate function per row.

`analytic_list_node` (in `xasl/xasl_analytic.hpp`) carries: function type, ORDER BY sort list, frame specification (`ROWS` / `RANGE`), accumulator.

`qdata_finalize_analytic_func(is_same_group = false)` resets the accumulator when a new partition boundary is detected (PARTITION BY key change).

> [!warning] query_analytic.cpp is C++ only
> Unlike most files in `src/query/` which are compiled as C++17 from `.c` files, `query_analytic.cpp` is explicitly `.cpp`. The three functions it exports use a C-linkage-compatible signature but internally use C++17 features.

## Hierarchy Aggregation

`qdata_evaluate_aggregate_hierarchy` handles aggregating MIN/MAX across a class hierarchy (superclass + subclasses each with their own B-tree). `HIERARCHY_AGGREGATE_HELPER` carries the array of `BTID`s and `HFID`s for each class in the hierarchy.

## Parallel Heap Scan Fast Path (BUILDVALUE_OPT)

> [!update] PR #7049 (`65d6915`, 2026-04-27)
> Parallel heap scan now supports a partial-aggregate fast path for **12 aggregate functions** in BUILDVALUE_PROC (single-tuple, no GROUP BY) queries: `COUNT_STAR`, `COUNT`, `MIN`, `MAX`, `SUM`, `AVG`, `STDDEV`, `STDDEV_POP`, `STDDEV_SAMP`, `VARIANCE`, `VAR_POP`, `VAR_SAMP`. Pre-7049 only `COUNT(*)` and `COUNT(col)` were supported (mode was named `COUNT_DISTINCT` then).

Each worker computes a partial accumulator independently; the main thread merges all partials via **`qdata_aggregate_accumulator_to_accumulator`** (the same merge primitive used by hash GROUP BY and serial aggregation — reused here to avoid per-aggregate logic duplication). MIN/MAX(DISTINCT) shortcuts to MIN/MAX (no-op DISTINCT). Other aggregates (`GROUP_CONCAT`, `MEDIAN`, `JSON_*`, bit aggregates, user SP) fall back to MERGEABLE_LIST or XASL_SNAPSHOT.

See [[components/parallel-heap-scan-result-handler]] for the worker-side accumulation switch and the heap-0 ↔ private-heap dance, and [[prs/PR-7049-parallel-buildvalue-heap]] for the full diff walkthrough.

## Related

- Parent: [[components/query|query]]
- [[components/query-executor|query-executor]] — calls `qdata_evaluate_aggregate_list` in GROUP BY loops; calls analytic functions after sort
- [[components/list-file|list-file]] — sort and spill storage for both aggregate and analytic paths
- [[components/partition-pruning|partition-pruning]] — provides `HIERARCHY_AGGREGATE_HELPER` for partitioned aggregate optimization
- [[components/parallel-heap-scan-result-handler]] — BUILDVALUE_OPT fast path: per-worker partial aggregation + accumulator merge
- [[prs/PR-7049-parallel-buildvalue-heap]] — PR that broadened the fast path
