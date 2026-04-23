---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/"
status: active
purpose: "Parallel execution subsystem for CUBRID queries: heap scan, hash join, sort, and subquery parallelism"
key_files:
  - "px_parallel.hpp / px_parallel.cpp (compute_parallel_degree — central degree selector)"
  - "px_worker_manager.hpp / px_worker_manager.cpp (per-query worker reservation)"
  - "px_worker_manager_global.hpp / px_worker_manager_global.cpp (singleton worker pool)"
  - "px_callable_task.hpp / px_callable_task.cpp (std::function-based task wrapper)"
  - "px_thread_safe_queue.hpp / px_thread_safe_queue.cpp (MPMC slot queue)"
  - "px_interrupt.hpp (interrupt + atomic_instnum + err_messages_with_lock)"
  - "px_sort.h / px_sort.c (parallel sort macros and API)"
  - "px_hash_join/ (subdir: build_partitions, execute_partitions)"
  - "px_heap_scan/ (subdir: templated manager + input/result/trace handlers)"
  - "px_query_execute/ (subdir: parallel subquery execution)"
public_api:
  - "parallel_query::compute_parallel_degree(type, num_pages, hint_degree)"
  - "worker_manager::try_reserve_workers(num_workers)"
  - "worker_manager::push_task(entry_task*)"
  - "worker_manager::wait_workers()"
  - "scan_open_parallel_heap_scan(...) / scan_start_parallel_heap_scan(...) / scan_next_parallel_heap_scan(...)"
  - "parallel_query::hash_join::build_partitions(...) / execute_partitions(...)"
  - "sort_start_parallelism(...) / sort_end_parallelism(...)"
tags:
  - component
  - cubrid
  - parallel
  - query
related:
  - "[[modules/src|src]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/parallel-worker-manager-global|parallel-worker-manager-global]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/parallel-interrupt|parallel-interrupt]]"
  - "[[components/parallel-hash-join|parallel-hash-join]]"
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-query-execute|parallel-query-execute]]"
  - "[[components/parallel-sort|parallel-sort]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[Query Processing Pipeline]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/query/parallel/` — Parallel Query Execution

This directory houses the entire parallel execution subsystem for CUBRID. It sits server-side (guarded by `SERVER_MODE` / `SA_MODE`) and adds thread-level parallelism to four query operations: heap scans, hash joins, external sort, and uncorrelated subqueries.

## Architecture Overview

```
parallel_query::compute_parallel_degree(type, num_pages, hint)
         │
         │  returns 0 = disable, N = degree
         ▼
worker_manager::try_reserve_workers(N)
         │  CAS loop on worker_manager_global::m_available
         │  returns nullptr on contention / insufficient workers
         ▼
 worker_manager  ──  worker_manager_global  ──  cubthread::worker_pool_type
 (per-query)          (process singleton)          (named "parallel-query")
         │
         ├─── px_hash_join/        build_partitions + execute_partitions
         ├─── px_heap_scan/        parallel_heap_scan::manager<RESULT_TYPE>
         ├─── px_query_execute/    parallel subquery execution
         └─── px_sort              SORT_EXECUTE_PARALLEL / SORT_WAIT_PARALLEL macros
```

---

## Class / Function Inventory — `px_parallel.{hpp,cpp}`

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `parallel_query::parallel_type` | enum class | `enum class parallel_type : int` | Tags the workload type — used to select the correct page threshold |
| `parallel_query::compute_parallel_degree` | free function | `UINT32(parallel_type, UINT64 num_pages, int hint_degree = -1) noexcept` | Central degree selector: reads system params once, applies formula, returns 0 to disable |

---

## `parallel_type` Enum — Complete Value List

Defined in `px_parallel.hpp` as `enum class parallel_type : int`:

| Enumerator | Value | Associated page-threshold param |
|------------|-------|---------------------------------|
| `HEAP_SCAN` | `0` | `PRM_ID_PARALLEL_HEAP_SCAN_PAGE_THRESHOLD` |
| `HASH_JOIN` | `1` | `PRM_ID_PARALLEL_HASH_JOIN_PAGE_THRESHOLD` |
| `SORT`      | `2` | `PRM_ID_PARALLEL_SORT_PAGE_THRESHOLD` |
| `SUBQUERY`  | `3` | No threshold — degree fixed at 1 (see below) |

---

## `compute_parallel_degree` — Full Breakdown

### One-time initialization (`std::call_once`)

```cpp
static std::once_flag once;
static std::size_t system_core_count;
static int parallelism;
static int heap_scan_page_threshold;
static int hash_join_page_threshold;
static int sort_page_threshold;

std::call_once(once, [] {
    parallelism          = prm_get_integer_value(PRM_ID_PARALLELISM);
    system_core_count    = cubthread::system_core_count();
    heap_scan_page_threshold = prm_get_integer_value(PRM_ID_PARALLEL_HEAP_SCAN_PAGE_THRESHOLD);
    hash_join_page_threshold = prm_get_integer_value(PRM_ID_PARALLEL_HASH_JOIN_PAGE_THRESHOLD);
    sort_page_threshold  = prm_get_integer_value(PRM_ID_PARALLEL_SORT_PAGE_THRESHOLD);
});
```

All six statics are initialised exactly once at first call, not at server startup. This is intentional — the system parameters must already be loaded before this function is first called.

> [!key-insight] `std::call_once` for lazy param caching
> Parameters are read lazily to avoid ordering problems during server boot. The `once_flag` is function-local (not class-level), so this is effectively a lock-free singleton for the parameter snapshot. Changing `PRM_ID_PARALLELISM` at runtime does NOT take effect because the statics are frozen after the first call.

> [!warning] Runtime parameter changes ignored
> Since all six parameters are cached in function-local statics via `std::call_once`, any dynamic change to `PRM_ID_PARALLELISM` or threshold parameters after server startup is silently ignored. A server restart is required.

### Decision flow

```
const UINT32 start_degree = 2

if system_core_count <= start_degree
    return 0  // disable: too few cores to bother

switch (type):
  SUBQUERY:
    auto_degree = 1  // fixed
    if hint < 0:       return MIN(1, parallelism)
    if hint >= 2:      return 1          // hint ignored, fixed degree
    else (0 or 1):     return 0          // hint disables it

  HEAP_SCAN / HASH_JOIN / SORT:
    page_threshold = MAX(per_type_threshold, start_degree)
    if num_pages < page_threshold: return 0   // below threshold

    hint handling:
      hint < 0 (auto):    compute logarithmic auto_degree
      hint >= 2:          return MIN(hint, min(num_pages, system_core_count))
      hint 0 or 1:        return 0  // hint disables it

    x = num_pages / page_threshold
    // GCC/Clang fast path:
    auto_degree = (63 - __builtin_clzll(x)) + start_degree
    // Portable fallback: manual MSB scan + start_degree

    return MIN(auto_degree, parallelism)
```

### Exact auto-degree formula

`auto_degree = floor(log2(num_pages / threshold)) + 2`

Implemented via `__builtin_clzll` (count leading zeros) on GCC/Clang for zero-overhead MSB detection. The portable fallback implements the same logic via a cascade of shift-and-compare steps. `start_degree = 2` is the floor — a parallel query always uses at least 2 workers when parallelism is enabled.

### Edge cases

| Condition | Return | Reason |
|-----------|--------|--------|
| `system_core_count <= 2` | 0 | Not worth parallelizing on dual-core |
| `num_pages < threshold` | 0 | Small scan — overhead > benefit |
| `SUBQUERY` with `num_pages != 0` | asserts | Caller bug: subquery has no page count |
| `hint_degree == 0` | 0 | Explicit disable via hint |
| `hint_degree == 1` | 0 | Below `start_degree` — treated as disable |
| unknown `type` | 0 + `assert_release_error` | Defensive fallback |

---

## Threading Model

- A **single named thread pool** (`"parallel-query"`) is owned by `worker_manager_global` (process singleton). Capacity = `PRM_ID_MAX_PARALLEL_WORKERS`.
- Each parallel query **reserves** N slots from the pool via an atomic CAS loop (non-blocking). Returns `nullptr` on contention.
- Tasks are `cubthread::entry_task` subclasses dispatched through `worker_manager_global::push_task()` → `cubthread::get_manager()->push_task(pool, task)`.
- Main thread **spins** on `m_active_tasks` in `wait_workers()` until all worker tasks call `pop_task()` (via `retire()`).

> [!warning] Spin wait in `wait_workers`
> `worker_manager::wait_workers()` uses `std::this_thread::yield()` in a busy-spin. This is intentional for short-lived parallel bursts but means the main thread is not sleeping. Avoid very high parallelism on systems with fewer physical cores.

---

## Lifecycle

```
server start
  └─ worker_manager_global::init()          (called once from boot)
       └─ std::call_once → thread_create_worker_pool("parallel-query", max_workers)

per-query
  └─ compute_parallel_degree(type, pages)    → degree N
       └─ worker_manager::try_reserve_workers(N)
            │  CAS on global m_available
            └─ returns worker_manager* or nullptr
                 │
                 ├─ push_task(callable_task*)   [×N]
                 ├─ tasks execute in pool workers
                 ├─ retire() → pop_task()       [×N]
                 └─ wait_workers() + release_workers()

server stop
  └─ worker_manager_global::destroy()       (asserts all returned)
```

---

## Build-Mode Guards

| File | Guard |
|------|-------|
| `px_parallel.hpp/cpp` | None — reads params/core count only, safe in all modes |
| All other files | `#if !defined(SERVER_MODE) && !defined(SA_MODE)` `#error Wrong module` |

---

## Constraints

- `parallelism` asserted `>= 0` and `<= system_core_count` at init time.
- `hint_degree` asserted `== -1` (auto) or `[0, PRM_MAX_PARALLELISM]`.
- `SUBQUERY` type requires `num_pages == 0` (asserted).
- All threshold parameters must be loaded before first call to `compute_parallel_degree`.

---

## Sub-Components

| Component | Source | Role |
|-----------|--------|------|
| [[components/parallel-worker-manager\|parallel-worker-manager]] | `px_worker_manager.*` | Per-query reservation handle |
| [[components/parallel-worker-manager-global\|parallel-worker-manager-global]] | `px_worker_manager_global.*` | Singleton pool |
| [[components/parallel-task-queue\|parallel-task-queue]] | `px_thread_safe_queue.*` + `px_callable_task.*` | Task dispatch primitives |
| [[components/parallel-interrupt\|parallel-interrupt]] | `px_interrupt.hpp` | Cross-thread interrupt + error propagation |
| [[components/parallel-sort\|parallel-sort]] | `px_sort.*` | Macro-based sort parallelism |
| [[components/parallel-hash-join\|parallel-hash-join]] | `px_hash_join/` | Hash join parallelism (separate ingest) |
| [[components/parallel-heap-scan\|parallel-heap-scan]] | `px_heap_scan/` | Heap scan parallelism (separate ingest) |
| [[components/parallel-query-execute\|parallel-query-execute]] | `px_query_execute/` | Subquery parallelism (separate ingest) |

---

## Related

- Parent: [[modules/src|src]]
- [[components/system-parameter|system-parameter]] — `PRM_ID_PARALLELISM`, `PRM_ID_MAX_PARALLEL_WORKERS`, three threshold params
- [[components/thread-manager|thread-manager]] — owns `cubthread::worker_pool_type`, called by global init/destroy
- [[components/storage|storage]] — heap files and page buffer accessed by parallel heap scan
- [[Query Processing Pipeline]]
- [[Build Modes (SERVER SA CS)]]
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
