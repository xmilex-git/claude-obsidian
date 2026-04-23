---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/query_executor.c"
status: active
purpose: "XASL tree walker and main query execution dispatch; entry point for all server-side SELECT, DML, and set operations"
key_files:
  - "query_executor.c (~27K lines)"
  - "query_executor.h (public API)"
  - "query_manager.c (query lifecycle, temp file management, result cache)"
  - "query_manager.h"
  - "stream_to_xasl.c (server-side XASL deserialization)"
  - "xasl_to_stream.c (client-side XASL serialization)"
  - "xasl_cache.h / xasl_cache.c (XASL plan cache)"
  - "subquery_cache.h / subquery_cache.c (uncorrelated subquery result cache)"
public_api:
  - "qexec_execute_query(thread_p, xasl, dbval_cnt, dbval_ptr, query_id) → QFILE_LIST_ID*"
  - "qexec_execute_mainblock(thread_p, xasl, xstate, lock_info) → int"
  - "qexec_execute_subquery_for_result_cache(thread_p, xasl, xstate) → int"
  - "qexec_start_mainblock_iterations(thread_p, xasl, xstate) → int"
  - "qexec_clear_xasl(thread_p, xasl, is_final, for_parallel_aptr) → int"
  - "qexec_insert_tuple_into_list(thread_p, list_id, outptr_list, vd, tplrec) → int"
  - "qexec_alloc_agg_hash_context_buildlist_xasl(...)"
  - "qexec_hash_gby_agg_tuple_public(...)"
tags:
  - component
  - cubrid
  - query
  - xasl
  - executor
related:
  - "[[components/query|query]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/list-file|list-file]]"
  - "[[components/aggregate-analytic|aggregate-analytic]]"
  - "[[components/partition-pruning|partition-pruning]]"
  - "[[components/memoize|memoize]]"
  - "[[Query Processing Pipeline]]"
  - "[[components/parallel-query|parallel-query]]"
created: 2026-04-23
updated: 2026-04-23
---

# `query_executor.c` — XASL Executor

The heart of CUBRID's server-side query evaluation. `query_executor.c` is ~27K lines and contains the XASL tree walker, all XASL node type handlers, DML execution, connect-by hierarchical queries, hash group-by, and inline parallel integration.

> [!key-insight] Single mega-file by design
> The CUBRID project treats large files (10K–30K lines) as intentional, not tech debt. Do not split `query_executor.c`. Use the internal function index to navigate.

## Execution Flow

```
qexec_execute_query()
  ├── Lookup / create XASL_STATE
  ├── Check XASL result cache (qfile_lookup_list_cache_entry)
  └── qexec_execute_mainblock()
        ├── BUILDLIST_PROC
        │     ├── qexec_start_mainblock_iterations()  (open scans)
        │     ├── Loop: scan_next_scan() → fetch tuple → evaluate predicate
        │     │         → qexec_insert_tuple_into_list()
        │     ├── GROUP BY: qfile_sort_list() + aggregate evaluation
        │     ├── Hash GROUP BY: qexec_alloc_agg_hash_context_buildlist_xasl()
        │     └── HAVING, DISTINCT, ORDER BY, LIMIT
        ├── BUILDVALUE_PROC
        │     └── Single-row aggregate (COUNT(*), MIN, MAX, etc.)
        ├── UNION_PROC / DIFFERENCE_PROC / INTERSECTION_PROC
        │     └── qfile_combine_two_list()
        ├── UPDATE_PROC / DELETE_PROC / INSERT_PROC
        │     └── Lock → heap op → index maintenance
        └── CONNECT_BY_PROC  (hierarchical query)
```

## Key State Structures

### `XASL_STATE`
```c
struct xasl_state {
  VAL_DESCR vd;        // value descriptor (DB_VALUE array, sys datetime, rands)
  QUERY_ID query_id;   // query associated with this XASL execution
  int qp_xasl_line;    // error line for diagnostics
};
```

### `VAL_DESCR`
Holds the host-variable `DB_VALUE` array, system datetime (evaluated once per query), and random seeds for `RAND()`/`LRAND()`. Shared across all nodes in one XASL tree execution.

## XASL Node Types Handled

| Proc Type | Description |
|-----------|-------------|
| `BUILDLIST_PROC` | Main SELECT — materialize tuples to list file |
| `BUILDVALUE_PROC` | Aggregate-only SELECT (no group keys) |
| `UNION_PROC` | UNION set operation |
| `DIFFERENCE_PROC` | EXCEPT / MINUS |
| `INTERSECTION_PROC` | INTERSECT |
| `SCAN_PROC` | Scan sub-node execution (called recursively) |
| `UPDATE_PROC` | Server-side UPDATE |
| `DELETE_PROC` | Server-side DELETE |
| `INSERT_PROC` | Server-side INSERT |
| `DO_PROC` | DO statement |
| `CONNECT_BY_PROC` | CONNECT BY hierarchical query |
| `MERGE_PROC` | MERGE (upsert) |

## Hash GROUP BY

> [!key-insight] Two-phase hash aggregate
> When group cardinality fits in memory, CUBRID uses an in-memory hash table (`AGGREGATE_HASH_CONTEXT` from `query_aggregate.hpp`). When the table overflows, partial accumulators are spilled to a list file, sorted by group key, and merged back. The spill threshold uses `HASH_AGGREGATE_VH_SELECTIVITY_THRESHOLD = 0.5` and `HASH_AGGREGATE_VH_SELECTIVITY_TUPLE_THRESHOLD = 2000` to detect high-selectivity cases that favor the sort path.

## Parallel Integration

`query_executor.c` includes `px_parallel.hpp` (server mode only) and calls `parallel_query::compute_parallel_degree()` to decide whether to hand off to a parallel scan or subquery executor. The non-parallel path is unchanged. See [[components/parallel-query]] for the parallel subsystem.

## XASL Cache

The XASL plan cache (`xasl_cache.c`) stores compiled `XASL_NODE` trees keyed by SQL hash text. Each cached entry has `XASL_CLONE` instances (one per concurrent execution). `qexec_clear_list_cache_by_class` invalidates result-cache entries when a class is modified.

## Subquery Cache

`subquery_cache.c` caches results of uncorrelated scalar subqueries within a single query execution. `qexec_execute_subquery_for_result_cache` is the entry used by the cache manager.

## Error Handling

```c
#define GOTO_EXIT_ON_ERROR \
  do { qexec_failure_line(__LINE__, xasl_state); goto exit_on_error; } while (0)
```
Every major operation uses this macro. Error context is stored in `xasl_state->qp_xasl_line` for diagnostics.

## Memoization Hook

`memoize.hpp` provides `memoize_get` / `memoize_put` — called within `qexec_execute_mainblock` to cache and reuse results of repeated subquery evaluations with the same input. See [[components/memoize]].

## Related

- Parent: [[components/query|query]]
- [[components/scan-manager|scan-manager]] — scan dispatch called by executor
- [[components/list-file|list-file]] — result materialization
- [[components/aggregate-analytic|aggregate-analytic]] — aggregate and window function evaluation
- [[components/partition-pruning|partition-pruning]] — partition prune before scan open
- [[components/memoize|memoize]] — subquery result caching
- [[components/parallel-query|parallel-query]] — parallel paths
- [[Query Processing Pipeline]]
