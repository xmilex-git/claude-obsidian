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
updated: 2026-05-11
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

> [!update] 2026-05-11 — Volcano 3-phase + scan_next_scan state machine (per [[sources/qp-analysis-executor]])
>
> ### 3-phase 구조 (`qexec_execute_mainblock`)
> ```
> Pre-processing
>   qexec_open_scan()              (main XASL + scan_ptr)
>   qexec_execute_mainblock(APTR)  // uncorrelated subquery 결과를 temp 에 1회 실행
>   alloc_agg_hash_context()
>
> Processing                       // 행 한 줄씩 산출 (Volcano iterator model)
>   qexec_intprt_fnc()
>   qexec_next_scan_block_iterations()
>     while scan_next_scan():
>       qexec_execute_mainblock(DPTR)   // correlated subquery, row 마다
>       qexec_execute_scan(SCAN_PTR)     // join inner
>       qexec_end_one_iteration()        // 1 row 결정/저장
>   qexec_end_mainblock_iterations()
>
> Post-processing
>   qexec_groupby()
>   qexec_execute_analytics()
>   qexec_orderby_distinct()
> ```
>
> ### `scan_next_scan()` 의 SUCCESS/END 상태머신
> - **SUCCESS** — predicate 까지 적용된 1 row 가 만들어진 상태. `for each row` 의 본문 진입.
> - **END** — 더 이상 조회 데이터 없음. post-processing 으로 이동.
>
> ### `scan_ptr` 와 sub-query 의 실행 경로 차이
> - `scan_ptr` (join inner) 는 mainblock 과 같이 처리됨 — `execute_mainblock` 안에서 `SCAN node → for each row → SCAN scan_ptr`.
> - `aptr` / `dptr` 의 subquery 는 **별도 main block 으로 독립 실행** — `execute_mainblock(aptr)` / `execute_mainblock(dptr)` 가 별도 호출됨.
> - **APTR (uncorrelated)** = pre-processing 에서 한번만 실행 → temp file. `XASL_LINK_TO_REGU_VARIABLE` flag set 시 일반 execute 경로 우회 → REGU 평가 시점 1회.
> - **DPTR (correlated)** = 매 outer row 마다 재실행.
>
> ### FROM 의 derived table vs SELECT/WHERE 의 subquery
> - **FROM 절 subquery** → `aptr` 으로 수행, 결과를 temp file 에 담음 → 그것을 `temp list scan` 으로 읽음.
> - **SELECT/WHERE 절 subquery** → `REGU_VARIABLE` 에 담기고 `fetch_peek_dbval` 흐름에서 실행됨.
>
> ### Block iteration (`qexec_next_scan_block_iterations`)
> 다음 access spec 처리를 위해 각 XASL block 을 초기화. Super class 를 `ALL` 로 조회하는 inheritance 케이스에서 핵심 — `(a∪b) ⋈ (c∪d)` 를 4개 block 으로 unfold.
>
> ### 1-row scan 내부
> ```
> scan_next_scan()
>   scan_next_heap_scan() (heap/index/list)
>     heap_next()                    // record read
>     eval_data_filter()             // predicate filter
>     heap_attrinfo_read_dbvalues()  // attribute → DB_VALUE
>     fetch_val_list()               // regu_var ← attr_info
>     eval_fnc()                     // expression evaluation
>     heap_attrinfo_read_dbvalues()
>     fetch_val_list()
> end_one_iteration()
>   qexec_generate_tuple_descriptor()
>   → write 1 row to temp file
> ```
>
> 분석서 핵심: predicate 사용 컬럼과 나머지를 구분하여 데이터를 regular variable 에 넣는다 — predicate fail 시 나머지 fetch 회피로 효율 향상.
>
> ### Post-processing — Group By 두 갈래
> - **Sort group by**: scan → result list → sort_listfile() → 정렬 list 순회하며 grouping → evaluation
> - **Hash group by**: in-memory hash ACC 누적 → memory overflow 시 partial list spill → 모든 row 처리 후 partial list 들 sort + merge
>
> 주요 함수:
> - Sort: `qexec_gby_get_next`, `qexec_gby_put_next`, `qdata_evaluate_aggregate_list`, `qdata_load_agg_hvalue_in_agg_list`
> - Hash: `qexec_hash_gby_get_next`, `qexec_hash_gby_put_next`, `qexec_hash_gby_agg_tuple` (`end_one_iteration` 후), `qdata_aggregate_accumulator_to_accumulator` (ACC-ACC 병합), `qdata_save_agg_htable_to_list` (overflow spill)
>
> ACCUMULATOR: 집계함수 1개당 1개. HASH ACCUMULATOR: 집계함수 갯수만큼 배열.
>
> ### Order by / distinct
> `sort_listfile()` 에 `SORT_DUP` 옵션:
> - `SORT_ELIM_DUP` — duplicate 제거 (DISTINCT)
> - `SORT_DUP` — duplicate 유지

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
- Source (internal R&D wiki): [[sources/qp-analysis-executor]] — 19-slide PPTX + hierarchical query + how-is-the-query-executed
