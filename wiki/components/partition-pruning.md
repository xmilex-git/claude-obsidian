---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/partition_sr.c"
status: active
purpose: "Runtime partition pruning: eliminate irrelevant partitions from access specs before scan open, and route DML to the correct partition"
key_files:
  - "partition_sr.c (server-side implementation)"
  - "partition_sr.h (PRUNING_CONTEXT, PRUNING_SCAN_CACHE, public API)"
  - "partition.h (MAX_PARTITIONS = 1024, shared client+server constants)"
public_api:
  - "partition_init_pruning_context(pinfo)"
  - "partition_load_pruning_context(thread_p, class_oid, pruning_type, pinfo) → int"
  - "partition_clear_pruning_context(pinfo)"
  - "partition_prune_spec(thread_p, vd, access_spec) → int"
  - "partition_prune_insert(thread_p, class_oid, recdes, scan_cache, pcontext, ...) → int"
  - "partition_prune_update(thread_p, class_oid, recdes, pcontext, ...) → int"
  - "partition_prune_unique_btid(pcontext, key, class_oid, class_hfid, btid) → int"
  - "partition_get_partition_oids(thread_p, class_oid, partition_oids, count) → int"
  - "partition_load_aggregate_helper(pcontext, spec, pruned_count, root_btid, helper) → int"
  - "partition_cache_init / partition_cache_finalize / partition_decache_class"
  - "partition_find_root_class_oid / partition_prune_partition_index"
tags:
  - component
  - cubrid
  - partition
  - query
  - pruning
related:
  - "[[components/query|query]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/aggregate-analytic|aggregate-analytic]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# `partition_sr.c` — Runtime Partition Pruning

Eliminates partitions that cannot contribute any rows to a query, and routes INSERT/UPDATE to the correct child partition. Runs server-side before scan objects are opened.

## Partition Types

CUBRID supports three partition strategies (tracked in `PRUNING_CONTEXT.partition_type`):

| `DB_PARTITION_TYPE` | Description |
|---------------------|-------------|
| Range partition | Partition key falls in a continuous range |
| List partition | Partition key equals one of an enumerated list of values |
| Hash partition | Partition key is hashed to a bucket |

Maximum partitions per table: **1024** (`MAX_PARTITIONS` in `partition.h`).

## PRUNING_CONTEXT

```c
struct pruning_context {
  THREAD_ENTRY *thread_p;
  OID root_oid;                 // partitioned root class
  REPR_ID root_repr_id;
  access_spec_node *spec;       // the access spec being pruned
  val_descr *vd;                // value descriptor for predicate eval
  DB_PARTITION_TYPE partition_type;

  OR_PARTITION *partitions;     // all partition descriptors
  OR_PARTITION *selected_partition; // if query targets a partition directly
  SCANCACHE_LIST *scan_cache_list;  // per-partition heap scan caches
  int count;                    // total partitions

  xasl_unpack_info *fp_cache_context;
  func_pred *partition_pred;    // the partition expression
  int attr_position;            // index key position of partition attr
  ATTR_ID attr_id;
  HEAP_CACHE_ATTRINFO attr_info;
  int error_code;
  int pruning_type;             // DB_PARTITIONED_CLASS or DB_PARTITION_CLASS
  bool is_attr_info_inited;
  bool is_from_cache;
};
```

## Pruning Flow

```
qexec_execute_mainblock()
  ↓
[for each access_spec in XASL node]
partition_prune_spec(thread_p, vd, access_spec)
  ├── partition_load_pruning_context()    (load from cache or schema)
  ├── Evaluate partition predicate with val_descr host variables
  ├── Match against each partition descriptor
  ├── Eliminate non-matching partitions from access_spec
  └── Result: access_spec->pruned_partition_list

scan_open_heap_scan() / scan_open_index_scan()
  └── Only opens scans on surviving partitions
```

## DML Routing

- `partition_prune_insert` — evaluates the new row's partition key expression against all partition ranges/lists/hash buckets to find the target `pruned_class_oid` and `pruned_hfid`.
- `partition_prune_update` — re-evaluates after the row is modified (partition key may have changed, requiring cross-partition move).
- `partition_prune_unique_btid` — maps a unique key lookup to the correct partition's B-tree.

## Partition Cache

`partition_cache_init` / `partition_cache_finalize` maintain a server-wide cache of `PRUNING_CONTEXT` objects keyed by class OID. `partition_decache_class` invalidates on DDL. `is_from_cache = true` indicates the context was borrowed from cache and must not be modified destructively.

## PRUNING_SCAN_CACHE

```c
struct pruning_scan_cache {
  HEAP_SCANCACHE scan_cache;       // cached heap scan for this partition
  bool is_scan_cache_started;
  func_pred_unpack_info *func_index_pred;  // function index predicates
  int n_indexes;
};
```
Multi-row DML statements (bulk INSERT, multi-row UPDATE) reuse `PRUNING_SCAN_CACHE` across rows to amortize heap scan cache setup cost.

## Aggregate Helper

`partition_load_aggregate_helper` fills `HIERARCHY_AGGREGATE_HELPER` with the `BTID`s and `HFID`s of all pruned partitions, enabling the aggregate optimizer (`qdata_evaluate_aggregate_optimize` in `query_aggregate.hpp`) to short-circuit full scans with index-only MIN/MAX reads across partitions.

> [!key-insight] Partition-aware MIN/MAX optimization
> When a query is `SELECT MIN(pk) FROM partitioned_table`, partition pruning combined with the aggregate optimizer can satisfy the result by reading a single leaf from one partition's index — O(1) rather than O(N).

## Related

- Parent: [[components/query|query]]
- [[components/query-executor|query-executor]] — calls `partition_prune_spec` before scan open
- [[components/scan-manager|scan-manager]] — opens scans on pruned partitions only
- [[components/aggregate-analytic|aggregate-analytic]] — aggregate optimizer uses partition hierarchy info
