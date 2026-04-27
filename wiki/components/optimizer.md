---
type: component
parent_module: "[[modules/src|src]]"
path: "src/optimizer/"
status: active
purpose: "Cost-based query planning"
key_files: []
public_api: []
tags:
  - component
  - cubrid
  - optimizer
  - query
related:
  - "[[modules/src|src]]"
  - "[[Query Processing Pipeline]]"
  - "[[components/parser|parser]]"
  - "[[components/query|query]]"
  - "[[components/xasl|xasl]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/optimizer/` — Cost-Based Query Planner

Selects join orders, access paths, and physical operators based on cost estimates. Output feeds [[components/xasl|XASL]] generation in [[components/parser]].

## Side of the wire

> [!key-insight] Client-side
> Like the parser, the optimizer runs on the **client** (`#if !defined(SERVER_MODE)`). The server receives only the chosen plan as serialized XASL.

## Inputs / outputs

- **Input:** annotated `PT_NODE` tree from [[components/parser]] (after name resolution + semantic check)
- **Output:** decisions consumed by `xasl_generation.c` to build the `XASL_NODE` plan

## Selectivity defaults

`query_planner.c` defines a set of `DEFAULT_*_SELECTIVITY` constants used whenever no statistics-backed estimate is available. These live in the `.c` as file-private `#define`s:

| Constant | Value | Used for |
|---|---|---|
| `DEFAULT_NULL_SELECTIVITY` | 0.01 | `IS NULL` when null stats unavailable |
| `DEFAULT_EXISTS_SELECTIVITY` | 0.1 | `EXISTS (subq)` |
| `DEFAULT_SELECTIVITY` | 0.1 | generic fallback |
| `DEFAULT_EQUAL_SELECTIVITY` | 0.001 | `ATTR = const` |
| `DEFAULT_EQUIJOIN_SELECTIVITY` | 0.001 | `ATTR = ATTR` across relations |
| `DEFAULT_COMP_SELECTIVITY` | 0.1 | `ATTR {<,<=,>,>=} const` |
| `DEFAULT_BETWEEN_SELECTIVITY` | 0.01 | `ATTR BETWEEN a AND b` |
| `DEFAULT_IN_SELECTIVITY` | 0.01 | `ATTR IN (...)` |
| `DEFAULT_RANGE_SELECTIVITY` | 0.1 | composite range terms |

`PRM_ID_LIKE_TERM_SELECTIVITY` (system parameter) drives the default for `LIKE` / `LIKE ESCAPE`. These constants are the bedrock cost-model assumptions — they are calibration-sensitive and rarely touched.

`PRED_CLASS { PC_ATTR, PC_CONST, PC_HOST_VAR, PC_SUBQUERY, PC_SET, PC_OTHER, PC_MULTI_ATTR }` and the file-local `qo_classify` helper partition predicate operands for the selectivity dispatch.

## Inputs / outputs, continued

Detail will be added on a deeper ingest of the optimizer source. AGENTS.md only lists this directory at one-line granularity.

## From the Manual (sql/tuning.rst, sql/parallel.rst — added 2026-04-27)

> [!gap] Documented contracts
> - **Full SQL HINT catalogue (28 hints)** is documented in `sql/tuning.rst`. See [[sources/cubrid-manual-sql-tuning-parallel]] for the table. Notable additions in 11.4: `USE_HASH`/`NO_USE_HASH` (HASH JOIN opt-in), `LEADING(t1 t2)` (finer than ORDERED).
> - **`SET OPTIMIZATION LEVEL n`** valid values: 1 (full, default), 2 (heuristic only), +256 (emit plan trace), +512 (emit join enumeration trace). So 1, 2, 257, 258, 513, 514.
> - **Plan cache regeneration** triggered ONLY when **BOTH**: (a) ≥6 minutes elapsed since last check, AND (b) `UPDATE STATISTICS` changed page count by ≥10× since plan was cached. (`sql/tuning.rst:14-20`).
> - **NEW 11.4 stat improvements**: sampling pages 1000 → **5000**; NDV (number of distinct values) collection improved; NOT LIKE selectivity added; function-based-index selectivity added; NDV duplicate weighting (>1% sample dup → reweight); `SSCAN_DEFAULT_CARD` to prevent inefficient NL JOIN on tiny cardinality estimates; LIMIT cost/cardinality reflected in plans.
> - **Index Skip Scan auto-selected in 11.4** — no `INDEX_SS` hint needed.
> - **Optimizer prefers cheaper index over PK** when applicable (NEW 11.4).
> - **Stored Procedure execution plans** now use index scans, eliminate unnecessary joins, and support result caching in correlated subqueries.

See [[sources/cubrid-manual-sql-tuning-parallel]] for the full hint catalogue and 11.4 optimizer changes.

## Related

- Parent: [[modules/src|src]]
- [[Query Processing Pipeline]]
- Source: [[cubrid-AGENTS]]
- Manual: [[sources/cubrid-manual-sql-tuning-parallel]]
