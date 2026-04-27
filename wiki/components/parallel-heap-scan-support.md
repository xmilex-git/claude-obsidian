---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_heap_scan/px_heap_scan_{checker,trace_handler}.{hpp,cpp}"
status: active
purpose: "Parallel heap scan supporting infrastructure: eligibility checker + aggregated trace / EXPLAIN ANALYZE stats"
key_files:
  - "px_heap_scan_checker.hpp (scan_check_parallel_heap_scan_possible)"
  - "px_heap_scan_trace_handler.{hpp,cpp} (child_stats struct, trace_storage_for_sibling_xasl class)"
public_api:
  - "scan_check_parallel_heap_scan_possible(XASL_NODE *xasl) — C linkage"
  - "parallel_heap_scan::trace_storage_for_sibling_xasl class"
tags:
  - component
  - cubrid
  - parallel
  - query
  - heap-scan
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-task|parallel-heap-scan-task]]"
  - "[[components/parallel-heap-scan-result-handler|parallel-heap-scan-result-handler]]"
  - "[[components/monitor|monitor]]"
  - "[[components/query-dump|query-dump]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan_checker` + `px_heap_scan_trace_handler` — Parallel Heap Scan Support

Two small modules bundled into one page because each is <300 lines and they serve the same hub ([[components/parallel-heap-scan]]).

## 1. Checker — `px_heap_scan_checker`

### Single public entry point

```cpp
extern "C" int scan_check_parallel_heap_scan_possible (XASL_NODE *xasl);
```

Returns non-zero degree when the XASL is eligible for parallel heap scan; 0 to fall back to serial. Called by the main thread **before** the manager is instantiated — a fast veto gate that avoids allocating the worker reservation machinery when parallelism wouldn't help.

### Eligibility signals (inferred from source + AGENTS.md context)

The checker consults:
- Page-count threshold (`PRM_ID_PARALLEL_HEAP_SCAN_PAGE_THRESHOLD`)
- XASL shape: sibling-scan list correctness, METHOD/SP presence, cross-thread dangerous ops
- Build mode: SERVER_MODE+SA_MODE only (enforced by parent module guard)
- `PRM_ID_PARALLELISM` global degree cap

### Internal flags + BUILDVALUE_OPT decision (post PR #7049)

> [!update] PR #7049 (`65d6915`, 2026-04-27)
> Renamed `count_opt` local + `CANNOT_COUNT_OPT` flag to `buildvalue_opt` / `CANNOT_BUILDVALUE_OPT` (same bit `0x1 << 2`). The check at the BUILDVALUE_PROC arm now calls `is_buildvalue_opt_supported_function(agg_it->function)` instead of the hardcoded `agg_it->function != PT_COUNT_STAR && agg_it->function != PT_COUNT`.

`possible_flags` (file-static enum, 3 bits):

- `CANNOT_PARALLEL_HEAP_SCAN` (`0x1 << 0`)
- `CANNOT_LIST_MERGE` (`0x1 << 1`)
- `CANNOT_BUILDVALUE_OPT` (`0x1 << 2`) — was `CANNOT_COUNT_OPT` pre-7049

`is_buildvalue_opt_supported_function` (file-static, `px_heap_scan_checker.cpp`) — returns true for `PT_COUNT_STAR`, `PT_COUNT`, `PT_MIN`, `PT_MAX`, `PT_SUM`, `PT_AVG`, `PT_STDDEV`, `PT_STDDEV_POP`, `PT_STDDEV_SAMP`, `PT_VARIANCE`, `PT_VAR_POP`, `PT_VAR_SAMP`. Anything outside this whitelist (e.g. `PT_GROUP_CONCAT`, `PT_MEDIAN`, `PT_JSON_*`, bit aggregates, user-defined SP) sets `CANNOT_BUILDVALUE_OPT` for the parent BUILDVALUE_PROC arm and falls back to `MERGEABLE_LIST` or `XASL_SNAPSHOT`. See [[prs/PR-7049-parallel-buildvalue-heap]].

If `!CANNOT_BUILDVALUE_OPT` after the full check, the checker calls `ACCESS_SPEC_SET_FLAG(specp, ACCESS_SPEC_FLAG_BUILDVALUE_OPT)` for every spec in the spec list — this flag is then read by `scan_open_parallel_heap_scan` to pick `RESULT_TYPE::BUILDVALUE_OPT`.

### Lifecycle

```
(main thread) query compile decides hint/auto degree
     │
     ├─► scan_check_parallel_heap_scan_possible(xasl)   O(n) over XASL tree
     │        │
     │        └─► 0       → fall back to serial scan_manager
     │        └─► N > 1   → [[components/parallel-query|compute_parallel_degree]] → manager
```

### Constraints

- `extern "C"`: called from C++ and C call-sites uniformly.
- Read-only: must not mutate `xasl`.
- Stateless: no global cache, no side effects.

---

## 2. Trace handler — `px_heap_scan_trace_handler`

### `child_stats`

Per-worker counters aggregated at the end of a parallel heap scan for EXPLAIN ANALYZE output:

```cpp
struct child_stats {
    UINT64 fetches;          // page fetches this worker did
    UINT64 ioreads;          // disk I/O reads
    UINT64 fetch_time;       // microseconds spent in page-buffer fetch
    UINT64 read_rows;        // raw rows read
    UINT64 qualified_rows;   // rows that passed the predicate
    struct timeval elapsed_time;
};
```

### `trace_storage_for_sibling_xasl` class

Jansson-based JSON accumulator. Each worker writes its `child_stats` into a shared structure under a mutex; the main thread serializes the aggregate as JSON at scan teardown.

### Purpose

Without this, parallel scan traces would look flat in EXPLAIN ANALYZE — no per-worker breakdown. With it, `[[components/query-dump|query-dump]]` can render per-sibling trees showing load balance, I/O hot spots, and qualification ratios.

### Lifecycle

```
manager constructor → trace_storage_for_sibling_xasl constructed
     │
     ├─► worker i: task::execute()
     │       │
     │       └─► trace_storage::add_stats(i, child_stats{...})
     │                │
     │                └─► mutex.lock → append to json array → unlock
     │
     └─► manager destructor → trace_storage::to_json() → output
```

### Constraints

- Mutex-protected because all workers write concurrently.
- Depends on [[dependencies/jansson|Jansson]] (2.10).
- Only enabled when the query runs under EXPLAIN ANALYZE / trace mode.

---

## Cross-references

- Hub: [[components/parallel-heap-scan|parallel-heap-scan]]
- Downstream: [[components/parallel-heap-scan-task|task]] writes stats here
- EXPLAIN rendering: [[components/query-dump|query-dump]]
- Monitoring parallel: [[components/monitor|monitor]], [[components/perfmon|perfmon]]
- Parent module: [[modules/src|src]]
