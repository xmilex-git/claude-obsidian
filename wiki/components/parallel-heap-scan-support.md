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
