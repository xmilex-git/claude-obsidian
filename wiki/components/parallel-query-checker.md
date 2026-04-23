---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_query_execute/px_query_checker.{hpp,cpp}"
status: active
purpose: "Eligibility gate for parallel subquery execution: checks whether an XASL subtree is safe to run in parallel workers"
key_files:
  - "px_query_checker.hpp (single C-linkage entry point)"
  - "px_query_checker.cpp (feasibility rules, XASL walk)"
public_api:
  - "check_parallel_subquery_possible(XASL_NODE *xasl) — extern \"C\""
tags:
  - component
  - cubrid
  - parallel
  - query
  - subquery
related:
  - "[[components/parallel-query-execute|parallel-query-execute]]"
  - "[[components/parallel-query-executor|parallel-query-executor]]"
  - "[[components/parallel-query-task|parallel-query-task]]"
  - "[[components/xasl|xasl]]"
  - "[[components/query-executor|query-executor]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_query_checker` — Parallel Subquery Feasibility Gate

Single-function module. Called before the parallel machinery is spun up for an uncorrelated subquery. Returns eligibility + recommended degree; callers fall back to serial if check fails.

## Public API

```cpp
extern "C" int check_parallel_subquery_possible (XASL_NODE *xasl);
```

Returns 0 when parallel is not safe or not beneficial; returns a positive degree hint when the checker greenlights the subquery.

## Why a checker exists

Not every XASL subtree is safe in a worker thread:
- XASL proc types with server-only side effects (DDL, connection-mutating ops) can't be cloned.
- Certain access methods (e.g., METHOD_SCAN invoking Java SPs) need the master session's thread-local context.
- Small result sets don't benefit from parallel overhead.
- MERGE / UPDATE / DELETE at the subquery level would break transactional semantics.

The checker encapsulates the rule set so `[[components/query-executor|query-executor]]` can query it once up front and cheaply fall back to the serial path if any rule fails.

## Execution path

```
query_executor (main) decides "subquery is correlated/not"
    │
    ├─► uncorrelated → check_parallel_subquery_possible(xasl)
    │       │
    │       │ rule sweep:
    │       │   - proc_type in whitelist?
    │       │   - access_spec safe?
    │       │   - depth / estimated rows / parallelism cap
    │       │
    │       ├─► 0 → serial subquery_cache path
    │       └─► N → [[components/parallel-query-executor]] with degree N
    │
    └─► correlated → always serial
```

## Constraints

- **Read-only**: must NOT mutate the XASL tree. The same tree is reused for the serial fallback.
- **`extern "C"`**: callable from C and C++ alike.
- **Fast**: O(n) walk over the XASL nodes in the subtree. No allocation on the happy path.
- **Build-mode**: SERVER_MODE + SA_MODE only (parent module guard).

## Lifecycle

Stateless, per-call. No cache, no globals.

## Related

- Hub: [[components/parallel-query-execute|parallel-query-execute]]
- Next stages: [[components/parallel-query-executor|parallel-query-executor]], [[components/parallel-query-task|parallel-query-task]]
- Feeds into: [[components/parallel-query|compute_parallel_degree]] (may further cap)
- Sibling checker (scan-level): [[components/parallel-heap-scan-support|parallel-heap-scan-support]]
