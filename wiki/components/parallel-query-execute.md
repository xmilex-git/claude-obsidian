---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_query_execute/"
status: developing
purpose: "Parallel execution of uncorrelated subqueries — degree fixed at 1 (main + 1 worker)"
key_files:
  - "px_query_execute/ (subdir — specific file names not fully enumerated in this ingest)"
public_api: []
tags:
  - component
  - cubrid
  - parallel
  - query
  - subquery
related:
  - "[[modules/src|src]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_query_execute/` — Parallel Subquery Execution

Handles parallel execution of **uncorrelated subqueries** in the XASL query plan. This is the most constrained parallelism mode: degree is intentionally fixed at 1 worker (main thread + 1 worker = 2 logical threads of work).

## Degree Policy

From `px_parallel.cpp` (`compute_parallel_degree`, case `SUBQUERY`):

```cpp
// degree fixed at 1 (main + gather = 2)
// to be revised when exact parallel count is available
// for many uncorrelated subqueries
auto_degree = 1;
```

> [!key-insight] Subquery parallelism is a stub, not full fan-out
> Unlike heap scan or hash join which can fan out to N workers, subquery parallel execution is fixed at degree 1. The TODO comment in `px_parallel.cpp` indicates the design intent is to eventually support parallel execution of many independent uncorrelated subqueries simultaneously, but it is not implemented yet.

> [!warning] hint_degree is ignored for subqueries
> If a SQL hint specifies a degree >= 2 for subquery parallelism, the code ignores the hint and uses degree 1 anyway. A hint of 1 explicitly disables parallel execution (returns 0).

## Integration Point

The subquery parallel path hooks into the XASL executor (`query_executor.c`) at points where uncorrelated `DPTR` or subquery XASLs are evaluated. The main thread evaluates the outer plan while the worker evaluates the subquery plan concurrently.

## File Structure

The `px_query_execute/` directory was not fully enumerated during this ingest. The subdir contains files related to subquery task dispatch and gather (collecting results from the worker back to the main thread). Follow-up reading of the subdir's file listing is recommended.

> [!warning] Incomplete ingest coverage
> Only the degree policy (from `px_parallel.cpp`) and the outer API shape are confirmed for this subcomponent. The internal task/gather implementation files were not resolved during this ingest pass. A follow-up ingest of `px_query_execute/*.hpp` files is recommended.

## Related

- [[components/parallel-query|parallel-query]] — degree selection and interrupt protocol
- [[components/parallel-worker-manager|parallel-worker-manager]] — task dispatch
- [[Query Processing Pipeline]] — where subqueries fit in XASL execution
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
