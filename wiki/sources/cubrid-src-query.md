---
type: source
title: "CUBRID src/query/ — XASL Execution Layer"
source_path: "src/query/"
ingest_date: 2026-04-23
status: complete
coverage: "84 top-level files; parallel/ subdir excluded (separately ingested)"
pages_created:
  - "[[components/query]]"
  - "[[components/query-executor]]"
  - "[[components/scan-manager]]"
  - "[[components/cursor]]"
  - "[[components/partition-pruning]]"
  - "[[components/dblink]]"
  - "[[components/list-file]]"
  - "[[components/aggregate-analytic]]"
  - "[[components/filter-pred-cache]]"
  - "[[components/memoize]]"
tags:
  - source
  - cubrid
  - query
  - xasl
  - ingest
related:
  - "[[components/query]]"
  - "[[components/parallel-query]]"
  - "[[Query Processing Pipeline]]"
---

# Source: `src/query/` XASL Execution Layer

## Coverage

Read: `AGENTS.md`, `query_executor.h`, `scan_manager.h`, `cursor.h`, `partition.h`, `partition_sr.h`, `dblink_2pc.h`, `dblink_scan.h`, `list_file.h`, `fetch.h`, `filter_pred_cache.h`, `query_aggregate.hpp`, `query_analytic.hpp`, `memoize.hpp`, `query_manager.h`, `xasl_cache.h`, top ~150 lines of `query_executor.c`, top ~180 lines of `scan_manager.c`.

Skipped: `parallel/` (already ingested as [[components/parallel-query]]), `string_opfunc.c`, `arithmetic.c`, `vacuum.c` (standalone subsystems, not core to execution dispatch).

## Pages Produced

| Page | Covers |
|------|--------|
| [[components/query]] | Hub: full subsystem map, execution pipeline diagram, type table |
| [[components/query-executor]] | `qexec_execute_mainblock` dispatch, XASL node types, hash GROUP BY, XASL cache |
| [[components/scan-manager]] | `SCAN_ID` union, all 15 scan types, `scan_next_scan` dispatch, index scan optimizations |
| [[components/cursor]] | Client-side cursor lifecycle, `CURSOR_ID` struct, copy vs. peek |
| [[components/partition-pruning]] | `PRUNING_CONTEXT`, prune-spec, DML routing, partition cache, aggregate helper |
| [[components/dblink]] | CCI-backed remote scan, 2PC protocol, `DBLINK_CONN_ENTRY` |
| [[components/list-file]] | Temp spool pages, sort, set ops, isolation-aware result cache |
| [[components/aggregate-analytic]] | Hash GROUP BY spill, sort-based GROUP BY, index MIN/MAX, window functions |
| [[components/filter-pred-cache]] | Claim/retire pattern for filtered index predicates |
| [[components/memoize]] | Hit-ratio self-disable, fixed-size allocator, `memoize_put_nullptr` |

## Key Insights

1. **`qexec_execute_mainblock` is the single entry point** for all server-side query execution. It dispatches on `XASL_NODE.type` covering SELECT (BUILDLIST/BUILDVALUE), all DML, set operations, and CONNECT BY — all in one ~27K-line file.

2. **`SCAN_ID` is a true polymorphic union** over 15 scan types including remote (dblink), JSON, method, parallel heap, and statistical sampling. The entire executor loop calls `scan_next_scan` without knowing the underlying storage.

3. **Hash GROUP BY has a two-phase spill path** using list files as overflow storage. The spill decision uses a 2000-tuple calibration window and a 50% selectivity threshold to choose between hash and sort aggregate paths at runtime.

4. **Memoize uses hit-ratio self-disable**: after 1000 misses it disables itself. This prevents the cache from wasting memory when every row has a unique subquery input (common in OLTP joins).

5. **Filter predicate cache uses exclusive lease (claim/retire)**, not shared read. Each scan holds exclusive ownership of its `pred_expr_with_context`, eliminating any need to lock the predicate tree during concurrent scan evaluation.

6. **Partition pruning feeds the aggregate optimizer**: `partition_load_aggregate_helper` supplies the full partition B-tree/heap hierarchy to `qdata_evaluate_aggregate_hierarchy`, enabling O(1) MIN/MAX across partitioned tables.

## Cross-References

- [[components/parallel-query]] — parallel extensions hook into `query_executor.c` via `px_parallel.hpp`; `S_PARALLEL_HEAP_SCAN` routes through `scan_manager.c`
- [[components/parser]] — parser/optimizer produce `XASL_NODE`; `xasl_to_stream.c` serializes; `stream_to_xasl.c` deserializes for this layer
- [[Query Processing Pipeline]] — this layer is the "Execution" stage

## Suggested Follow-ups

1. **`query_opfunc.c`** — built-in scalar and aggregate function implementations (SUM, AVG, RANK, etc.) — not fully read; worth a dedicated `query-opfunc.md` page.
2. **`string_opfunc.c` (~28K lines)** — all string functions; huge surface area for correctness bugs.
3. **`vacuum.c`** — MVCC garbage collection lives in `src/query/`; architecturally belongs with transaction/MVCC but its placement here is surprising and worth documenting.
4. **`query_hash_scan.c`** — hash join scan implementation; complements [[components/parallel-hash-join]] for the non-parallel path.
5. **`xasl_cache.c`** — plan cache lifecycle (LRU eviction, clone management, invalidation); XASL cache and list-file result cache interact.
