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
  - "partition_get_scancache(pcontext, partition_oid) → PRUNING_SCAN_CACHE*"
  - "partition_new_scancache(pcontext) → PRUNING_SCAN_CACHE*"
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
  - "[[components/btree|btree]]"
  - "[[components/heap-file|heap-file]]"
  - "[[Query Processing Pipeline]]"
  - "[[Build Modes (SERVER SA CS)]]"
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

## SELECT vs DML Pruning Differences

SELECT and DML use the same `PRUNING_CONTEXT` / `partition_prune_spec` mechanism but diverge in several ways:

| Dimension | SELECT | INSERT / UPDATE |
|---|---|---|
| Entry point | `partition_prune_spec(thread_p, vd, access_spec)` | `partition_prune_insert` / `partition_prune_update` |
| Evaluated against | Constant-folded WHERE predicates in `val_descr` | Actual row values from `RECDES` record |
| Number of partitions | All matching partitions (range/list: subset; hash: single) | Exactly one target partition |
| Cross-partition move | N/A | UPDATE re-evaluates; if partition key changed, old partition is deleted and new partition is inserted |
| Scan cache reuse | `HEAP_SCANCACHE` opened per surviving partition | `PRUNING_SCAN_CACHE` reused across multiple rows in bulk DML |

### `partition_prune_spec` detail
- Loads `PRUNING_CONTEXT` from server cache or schema.
- Evaluates `pcontext->partition_pred` (the partition expression function predicate) using `pcontext->attr_info`.
- For RANGE: binary-searches partition ranges; any partition whose range overlaps the predicate's possible values survives.
- For LIST: walks enumerated values; only partitions with a matching value survive.
- For HASH: evaluates the hash function and identifies the single target bucket.
- Updates `access_spec->pruned_partition_list` — only surviving partitions are opened by `scan_open_heap_scan` / `scan_open_index_scan`.

## Partition Cache

The server maintains a global cache of `PRUNING_CONTEXT` objects keyed by class OID (`partition_cache_init` / `partition_cache_finalize`):
- `partition_load_pruning_context` checks the cache first; cache hit sets `pinfo->is_from_cache = true`.
- `partition_decache_class` is called on DDL changes (ALTER TABLE PARTITION, DROP TABLE) to invalidate stale contexts.
- Cached contexts must not be destructively modified — callers that need to modify the context copy it or work within the `pinfo->is_from_cache` constraint.

> [!warning] Context ownership
> When `is_from_cache = true`, `partition_clear_pruning_context` does **not** free the cached data. Only `partition_cache_finalize` at server shutdown reclaims those resources. Callers that create their own context (not from cache) must call `partition_clear_pruning_context` explicitly.

## PRUNING_SCAN_CACHE and Multi-Row DML

For bulk INSERT and multi-row UPDATE, opening a new `HEAP_SCANCACHE` per row would be prohibitively expensive. `PRUNING_SCAN_CACHE` amortises this:

```c
struct pruning_scan_cache {
  HEAP_SCANCACHE scan_cache;           // cached heap scan for this partition
  bool is_scan_cache_started;
  func_pred_unpack_info *func_index_pred; // function index predicates (one per functional index)
  int n_indexes;
};
```

`partition_get_scancache(pcontext, partition_oid)` returns an existing cache if one is open for this partition; `partition_new_scancache(pcontext)` allocates a fresh one appended to `pcontext->scan_cache_list`. All caches in the list are closed and freed by `partition_clear_pruning_context`.

## Aggregate Optimization (O(1) MIN/MAX)

`partition_load_aggregate_helper` fills `HIERARCHY_AGGREGATE_HELPER` for the aggregate optimizer:

```c
// partition_sr.h
extern int partition_load_aggregate_helper(
  PRUNING_CONTEXT *pcontext,
  access_spec_node *spec,
  int pruned_count,
  BTID *root_btid,
  HIERARCHY_AGGREGATE_HELPER *helper
);
```

The helper records the `BTID` and `HFID` for each surviving partition. When `qdata_evaluate_aggregate_optimize` (in `query_aggregate.hpp`) detects a `MIN` or `MAX` aggregate over a partition key with a B-tree index, it:
1. Gets the helper from `partition_load_aggregate_helper`.
2. For each partition's B-tree, reads only the first (for MIN) or last (for MAX) leaf entry.
3. Takes the best value across all partitions.

> [!key-insight] O(1) MIN/MAX across partitioned table
> `SELECT MIN(pk) FROM partitioned_table` after pruning can be satisfied by reading a single B-tree leaf per surviving partition — each read is O(log N) where N is rows in that partition, but the aggregation over partition results is O(k) where k is the number of surviving partitions. Combined with aggressive pruning (e.g. `WHERE part_col = 5` on a list-partitioned table), k=1 and the query cost is O(log N) total instead of O(N).

## 2-Phase Pruning (Static + Dynamic)

Pruning operates in two phases:

**Phase 1 — Static (query compile, client-side)**: The optimizer recognises constant predicates on the partition expression and can eliminate partitions entirely from the XASL plan before serialisation. This is done in `src/optimizer/` (not in `partition_sr.c`).

**Phase 2 — Dynamic (server-side, at `qexec_execute_mainblock`)**: `partition_prune_spec` is called with the current `val_descr` at execution time, allowing pruning on host variables and session parameters that are only known at runtime (e.g. `WHERE part_col = ?`). This is the runtime pruning implemented in `partition_sr.c`.

> [!key-insight] Two-phase pruning
> Static pruning happens client-side during plan generation; the XASL stream may already contain only a subset of partitions. Dynamic pruning re-evaluates at execution time using actual bound parameter values. Both phases can be active on the same query — static eliminates what's known at compile time; dynamic finishes the job at runtime.

## Constraints

- **Server-mode only**: `partition_sr.h` enforces `#if !defined(SERVER_MODE) && !defined(SA_MODE) #error Belongs to server module`.
- **MAX_PARTITIONS = 1024**: Hard limit in `partition.h`; attempting to define more partitions will fail at DDL time.
- **Partition key type**: The partition expression must be a single scalar function predicate (`func_pred`). No multi-column partition keys.
- **`attr_id` / `attr_position`**: `PRUNING_CONTEXT` tracks a single `ATTR_ID` (the partition attribute). For index-based pruning, `attr_position` is its position in the index key.
- **Global indexes**: The commented-out `partition_is_global_index` (`#if 0`) suggests global secondary indexes were planned or partially implemented but are currently disabled.

## Lifecycle

```
server start  : partition_cache_init() — allocate LF_HASH_TABLE for PRUNING_CONTEXT cache
DDL           : partition_decache_class(thread_p, class_oid) — invalidate on ALTER/DROP
query start   : partition_init_pruning_context(pinfo) — zero-fill
load          : partition_load_pruning_context(thread_p, class_oid, pruning_type, pinfo)
                  — cache lookup or schema load; sets partition_type, partitions[], count
prune SELECT  : partition_prune_spec(thread_p, vd, access_spec)
                  — evaluate pred; update access_spec->pruned_partition_list
prune INSERT  : partition_prune_insert(thread_p, class_oid, recdes, ..., pruned_class_oid, pruned_hfid)
prune UPDATE  : partition_prune_update(thread_p, class_oid, recdes, ...)
agg opt       : partition_load_aggregate_helper(pcontext, spec, pruned_count, root_btid, helper)
cleanup       : partition_clear_pruning_context(pinfo) — close scan caches; free if not from cache
server stop   : partition_cache_finalize() — free all cached contexts
```

## Related

- Parent: [[components/query|query]]
- [[components/query-executor|query-executor]] — calls `partition_prune_spec` before scan open
- [[components/scan-manager|scan-manager]] — opens scans only on partitions surviving pruning
- [[components/aggregate-analytic|aggregate-analytic]] — aggregate optimizer consumes `HIERARCHY_AGGREGATE_HELPER`
- [[components/btree|btree]] — `partition_prune_unique_btid` and `partition_prune_partition_index` resolve B-tree IDs per partition
- [[components/heap-file|heap-file]] — `HEAP_SCANCACHE` embedded in `PRUNING_SCAN_CACHE`
- [[Build Modes (SERVER SA CS)]] — server/SA only; compile-time guard in `partition_sr.h`
