---
created: 2026-04-27
type: source
title: "CUBRID Manual — Tuning + Parallel Execution"
source_path: "/home/cubrid/cubrid-manual/en/sql/tuning.rst, tuning_index.rst, join_method.inc, parallel.rst, parallel_index.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - sql
  - tuning
  - optimizer
  - parallel
  - hints
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-sql-foundation]]"
  - "[[sources/cubrid-manual-sql-dml]]"
  - "[[components/optimizer]]"
  - "[[components/optimizer-rewriter]]"
  - "[[components/parallel-query]]"
  - "[[components/parallel-hash-join]]"
  - "[[components/scan-hash]]"
  - "[[components/xasl-cache]]"
  - "[[components/subquery-cache]]"
---

# CUBRID Manual — Tuning + Parallel Execution

**Ingested:** 2026-04-27
**Source files:** `sql/tuning.rst` (4925), `tuning_index.rst` (10), `join_method.inc` (~1100, included by tuning.rst), `parallel.rst` (724), `parallel_index.rst` (12). **Total ~5.7K lines.**

## What This Covers

The **performance section** of the SQL reference: how the optimizer reasons, the full SQL hint catalogue, plan dumps and trace, sort/limit/skip-scan/loose-scan optimizations, parallel execution rules, hash-join methods, and the subquery cache.

## Section Map

### tuning.rst (4925 lines — the densest manual file)
- **UPDATE STATISTICS** — sampling 5000 pages by default in 11.4 (was 1000); FULLSCAN option
- **`;info stats`** output format
- **Plan dumps** — `SET OPTIMIZATION LEVEL` (1/2/257/258/513/514)
- **Optimizer Principle** — selectivity, expected rows, sequential vs index scan cost, join cost
- **Query Profiling** — `SET TRACE ON`, `SHOW TRACE`, JSON output
- **Full SQL HINT catalogue** (28 hints — see below)
- **USING/USE/FORCE/IGNORE INDEX** clauses
- **Filtered indexes**, **function-based indexes**
- **Covering index** (with VARCHAR truncation gotcha)
- **ORDER BY / GROUP BY skip optimization**
- **Descending index scan**
- **Multi-key range optimization** (`multi_range_optimization_limit`, default 1000)
- **Index skip scan (ISS)**
- **Loose index scan** (cardinality > 100x)
- **In-memory sort + sort-limit** (`sort_limit_max_count`)
- **Join-elimination** (outer→inner conversion, FK/PK inner-join elimination, LEFT OUTER elimination)
- **View merging**, **predicate push**, **subquery unnest** (IN-only), **transitive predicate**
- **QUERY_CACHE + subquery cache** (correlated cache, default 2 MB, hit-ratio < 90% disables)
- **Plan-cache regen rules** (6 min + 10× page size change)

### join_method.inc (~1100 lines — included into tuning.rst)
- Nested Loop (with index-join, hash list scan)
- Sort-Merge (`optimizer_enable_merge_join`)
- **Hash Join** (`USE_HASH`/`NO_USE_HASH`, build/probe selection)
- **Hash methods**: memory / hybrid / file / skip — selected by `max_hash_list_scan_size` (default 8 MB)
- **Outer-join build-side**: opposite of join direction (LEFT OUTER → right builds)

### parallel.rst (724 lines)
- **Parallel heap/list/index scan** (sector-partitioned for heap/list; shared cursor for index)
- **Parallel subquery** (degree fixed at 2)
- **Parallel hash join**
- **Parallel sort**
- **Throughput rule table**: 2048 pages → degree 2; doubling per +1; capped MIN with `parallelism`
- **`max_parallel_workers` (default 100)**, **`parallelism` (default 4, max MIN(32, cores))**
- **Hints**: `PARALLEL(degree)`, `NO_PARALLEL_SCAN`, `NO_PARALLEL_SUBQUERY`
- **Result-collection modes**: mergeable list / buildvalue / row-by-row
- **BUILDVALUE optimization aggregates**: COUNT/MIN/MAX/SUM/AVG/STDDEV*/VARIANCE*
- **Per-flavor constraints**: no ISS/ILS/KEYLIMIT/desc-index/filtered-index/min_max for parallel index; no row-by-row for parallel list; sort-merge auxiliary subtrees disqualified
- **SQL-trace fields**: `parallel workers`, `gather`, per-worker min..max ranges

## Full SQL Hint Catalogue (28 hints)

| Hint | Effect |
|---|---|
| `USE_NL` | Force nested-loop join |
| `USE_MERGE` | Force sort-merge join |
| `USE_HASH` | Force hash join (NEW 11.4) |
| `NO_USE_HASH` | Forbid hash join |
| `ORDERED` | Use FROM-clause order as join order |
| `LEADING(<tables>)` | Specify prefix tables in join order (NEW 11.4 — finer than ORDERED) |
| `USE_IDX(<table>.<idx>)` | Force specific index |
| `USE_DESC_IDX` | Force descending index scan |
| `USE_SBR` | Use subquery-based rewrite |
| `INDEX_SS` | Force Index Skip Scan |
| `INDEX_LS` | Force Loose Index Scan (cardinality > 100x) |
| `NO_DESC_IDX` | Disallow descending index |
| `NO_COVERING_IDX` | Disallow covering scan (escape covering-VARCHAR-truncation gotcha) |
| `NO_MULTI_RANGE_OPT` | Disable multi-key range optimization |
| `NO_SORT_LIMIT` | Disable sort+limit fusion |
| `NO_SUBQUERY_CACHE` | Disable correlated subquery cache for this subquery |
| `NO_PUSH_PRED` | Disable predicate push-down |
| `NO_MERGE` | Disable merge join (synonym of NO_USE_MERGE?) |
| `NO_ELIMINATE_JOIN` | Disable join-elimination optimizations |
| `NO_HASH_AGGREGATE` | Disable hash GROUP BY (force sort-based) |
| `NO_HASH_LIST_SCAN` | Disable hash list scan |
| `NO_LOGGING` | Skip log records (loaddb-style; very dangerous in user queries) |
| `PARALLEL(<degree>)` | Force parallel degree |
| `NO_PARALLEL_SCAN` | Disable parallel scan |
| `NO_PARALLEL_SUBQUERY` | Disable parallel subquery |
| `RECOMPILE` | Force re-plan (bypass plan cache) |
| `QUERY_CACHE` | Cache result of this query |
| (also `MERGE`'s `USE_UPDATE_IDX` / `USE_INSERT_IDX`) |

## Key Facts

### Optimizer levels
- `SET OPTIMIZATION LEVEL n` where n ∈ {1, 2, 257, 258, 513, 514}.
- 1 — full optimization (default).
- 2 — heuristic only (skip cost-based).
- +256 — emit plan trace to log.
- +512 — emit join enumeration trace.

### Plan cache regeneration
- Triggered ONLY when BOTH:
  1. ≥6 minutes elapsed since last check, AND
  2. `UPDATE STATISTICS` changed page count by ≥10× since plan was cached.
- `cubrid plandump` exposes cached plans.

### Subquery cache (correlated)
- Default 2 MB.
- **Auto-disables mid-query if hit-ratio < 90%** OR memory exceeds limit.
- Trace shows `status: enabled | disabled`.
- Disqualified for: random functions, sys_guid, CONNECT BY, OID features.
- **Extended in 11.4** — now applies in CTE non-recursive parts and uncorrelated subqueries (with `/*+ query_cache */` hint).

### Sort-Limit
- `sort_limit_max_count` default **1000**.
- Triggered when `ORDER BY ... LIMIT n` and `n` ≤ threshold.
- 11.4: bind-variable + expression LIMIT now eligible.

### Hash Join (NEW 11.4)
- Opt-in via `/*+ USE_HASH */`.
- Hash methods (selected dynamically by `max_hash_list_scan_size`, default 8 MB):
  - **memory** — fits in memory
  - **hybrid** — partitioned spill
  - **file** — temp-file based
  - **skip** — empty-side optimization
- Below 12 K rows typically forces file-hash.
- Outer-join build-side: opposite of join direction (LEFT OUTER → right table builds the hash).
- Trace shows `gather: hash temp(m|h|f)`.

### Index Skip Scan (ISS)
- For composite index `(A, B)` with predicate `B = ?` (no A predicate).
- Auto-selected without `INDEX_SS` hint **since 11.4** (improvement).

### Loose Index Scan (ILS)
- For `SELECT DISTINCT col1` from `(col1, col2)` index.
- Cardinality threshold > 100x.

### Covering Index VARCHAR truncation gotcha
- Covering scan returns **trailing-space-stripped values** for VARCHAR.
- Use `NO_COVERING_IDX` hint to force heap fetch and get the true value.

### Join elimination
- **Inner-join FK→PK elimination**: equality joins only, all join cols covered, PK/UNIQUE on right table, no other refs.
- **LEFT OUTER elimination**: PK/UNIQUE on right, no other refs.

### Parallel scan rules (parallel.rst)
- **Hard floor**: input must have ≥ **2048 pages** (~32 MB at 16 K page size). Below that, even explicit `PARALLEL` hint is ignored.
- Throughput rule: 2048 pages → degree 2, doubling per +1 up to 4 M pages.
- Capped at MIN(`parallelism`, 32, cores).

### Parallel subquery
- **Degree fixed at 2** regardless of subquery count.
- Uncorrelated only.
- Disqualified: correlated subqueries, CTE recursion, cross-references.

### Parallel hash join skip case
- Outer-join with empty outer input → skip method (no work).

## NEW 11.4 Performance Improvements

- **Sampling pages 1000 → 5000** for UPDATE STATISTICS.
- **Improved NDV (number of distinct values) collection**.
- **Reduced rule-based optimization**, more cost-based.
- **Index Filter Scan selectivity** added to cost model.
- **NOT LIKE selectivity** added.
- **Function-based index selectivity** added.
- **NDV duplicate weighting** (>1% sample dup → reweight stats).
- **SSCAN_DEFAULT_CARD** introduced — prevents inefficient NL JOIN plans on tiny cardinality estimates.
- **LIMIT cost/cardinality** reflected in plans.
- **Eliminated redundant join conditions** more aggressively.
- **Removed unnecessary INNER JOIN** — additional join elimination.
- **Trace `fetch_time` field added**.
- **Index Skip Scan auto-selected** (no need for `index_ss` hint).
- **Optimizer prefers better index over Primary Key** when cheaper.
- **Stored Procedure execution plans** now use index scans, eliminate unnecessary joins, and support result caching in correlated subqueries.
- **Concurrency**: unnecessary X-LOCKs released after all conditions evaluated.

## Cross-References

- [[components/optimizer]] — XASL generation, cost model
- [[components/optimizer-rewriter]] — optimizer rewriting hub
- [[components/optimizer-rewriter-select]] — join elimination
- [[components/optimizer-rewriter-subquery]] — subquery unnest, predicate push
- [[components/optimizer-rewriter-term]] — predicate term rewriting
- [[components/parallel-query]] — parallel scan, throughput rules
- [[components/parallel-hash-join]] — hash join methods
- [[components/parallel-query-execute]] — parallel subquery
- [[components/scan-hash]] — hash list scan vs hash join
- [[components/xasl-cache]] — plan cache regeneration
- [[components/subquery-cache]] — correlated subquery cache
- [[components/query-regex]] — `regexp_engine` parameter

## Incidental Wiki Enhancements

- [[components/optimizer]]: documented full 28-hint SQL HINT catalogue, plan-cache regeneration rule (6 min + 10× page-size change), `SET OPTIMIZATION LEVEL` 1/2/+256 trace/+512 join-enum levels.
- [[components/optimizer-rewriter]]: documented SORT-LIMIT optimization conditions (`sort_limit_max_count` 1000 default; FK→PK joins; LEFT/RIGHT outer driving table eligibility); 11.4 bind-var/expression LIMIT now eligible.
- [[components/optimizer-rewriter-subquery]]: documented subquery unnest = IN-only (not EXISTS); transitive predicate; predicate push exclusion list (CONNECT BY, aggregates/analytics, ROWNUM, methods, RANDOM/SYS_GUID, OUTER JOIN with NULL-transformation).
- [[components/optimizer-rewriter-select]]: documented inner-join FK→PK elimination + LEFT OUTER elimination conditions (PK/UNIQUE on right table, equality joins only, all join cols covered, no other refs).
- [[components/parallel-query]]: documented hard floor 2048 pages for parallel scan; throughput rule table 2048 → degree 2, doubling; capped MIN(`parallelism`=4, 32, cores).
- [[components/parallel-hash-join]]: documented 4 hash methods (memory/hybrid/file/skip), `max_hash_list_scan_size` 8 MB default selector; outer-join build-side = opposite of join direction; trace `gather: hash temp(m|h|f)`.
- [[components/parallel-query-execute]]: documented parallel subquery degree fixed at 2; uncorrelated only; correlated/CTE-recursive disqualified.
- [[components/xasl-cache]]: documented plan cache regeneration trigger (6 minutes elapsed AND ≥10× page-size delta with stats update).
- [[components/subquery-cache]]: documented default 2 MB cache size, hit-ratio < 90% auto-disable; correlated-only baseline (extended in 11.4 to CTE/uncorrelated via `/*+ query_cache */`); disqualifying functions (random, sys_guid, CONNECT BY, OID features); `RESULT CACHE (reference count: N)` trace line.
- [[components/scan-hash]]: documented hash list scan vs hash join distinction; `gather: hash temp(m|h|f)` profiling field.
- [[components/query-regex]]: documented `regexp_engine` parameter — Google RE2 default since 11.2; C++ `<regex>` deprecated; Spencer engine removed in 11.0; POSIX `[[:<:]]/[[:>:]]` no longer accepted (use `\b`).

## Key Insight

Tuning in CUBRID is **hint-driven**: 28 documented hints, four hash-join methods, and a complex parallel-execution rule table. The **most-cited gotcha** is the covering-index VARCHAR truncation (use `NO_COVERING_IDX` to escape). The **biggest 11.4 wins** are: HASH JOIN opt-in, LEADING hint (finer than ORDERED), expanded subquery cache (CTE + uncorrelated), and auto-selected Index Skip Scan. Parallel scan needs ≥2048 pages or it's silently single-threaded; parallel subquery is degree-2 fixed regardless of subquery count.
