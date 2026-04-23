---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/"
status: developing
purpose: "Parallel execution subsystem for CUBRID queries: heap scan, hash join, sort, and subquery parallelism"
key_files:
  - "px_parallel.hpp / px_parallel.cpp (compute_parallel_degree — central degree selector)"
  - "px_worker_manager.hpp / px_worker_manager.cpp (per-query worker reservation)"
  - "px_worker_manager_global.hpp / px_worker_manager_global.cpp (singleton worker pool)"
  - "px_callable_task.hpp (std::function-based task wrapper)"
  - "px_thread_safe_queue.hpp (MPMC slot queue)"
  - "px_interrupt.hpp (interrupt + atomic_instnum + err_messages_with_lock)"
  - "px_sort.h (parallel sort macros and API)"
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
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/parallel-hash-join|parallel-hash-join]]"
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-query-execute|parallel-query-execute]]"
  - "[[components/parallel-sort|parallel-sort]]"
  - "[[Query Processing Pipeline]]"
  - "[[components/storage|storage]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/query/parallel/` — Parallel Query Execution

This directory houses the entire parallel execution subsystem for CUBRID. It sits server-side (guarded by `SERVER_MODE` and `SA_MODE`) and adds thread-level parallelism to four query operations: heap scans, hash joins, external sort, and uncorrelated subqueries.

## Architecture Overview

```
parallel_query::compute_parallel_degree()
         │
         │  (returns 0 = disable, N = degree)
         ▼
worker_manager::try_reserve_workers(N)
         │
         │  (borrows N slots from global singleton pool)
         ▼
 worker_manager_global  ──── cubthread::worker_pool_type
         │                         (named "parallel-query")
         │
         ├─── px_hash_join/        build_partitions + execute_partitions
         │        task_manager → split_task / join_task (HASHJOIN_MANAGER)
         │
         ├─── px_heap_scan/        parallel_heap_scan::manager<RESULT_TYPE>
         │        input_handler_ftabs (page-set distribution)
         │        result_handler<T>   (write per-thread, read by main)
         │        task<T>             (XASL clone per worker)
         │
         ├─── px_query_execute/    parallel subquery execution
         │
         └─── px_sort              SORT_EXECUTE_PARALLEL / SORT_WAIT_PARALLEL macros
```

## Parallel Types

Defined in `px_parallel.hpp` as `parallel_query::parallel_type`:

| Enum | Use | Page-threshold param |
|------|-----|----------------------|
| `HEAP_SCAN` | Full heap scan | `PRM_ID_PARALLEL_HEAP_SCAN_PAGE_THRESHOLD` |
| `HASH_JOIN` | Hash join build + probe | `PRM_ID_PARALLEL_HASH_JOIN_PAGE_THRESHOLD` |
| `SORT` | External sort | `PRM_ID_PARALLEL_SORT_PAGE_THRESHOLD` |
| `SUBQUERY` | Uncorrelated subqueries | Fixed degree = 1 (main + 1 worker) |

## Degree Selection — `compute_parallel_degree`

> [!key-insight] Logarithmic auto-degree
> Degree grows logarithmically with page count: `degree = floor(log2(num_pages / threshold)) + 2`. Uses `__builtin_clzll` on GCC/Clang. Returns 0 (disable) when the system has ≤ 2 cores or page count is below threshold.

Steps:
1. Read `PRM_ID_PARALLELISM` (system max degree) and `cubthread::system_core_count()` once (via `std::call_once`).
2. Check page count against the per-type threshold.
3. If `hint_degree` ≥ 0 (SQL-hint override) use it directly (capped at core count).
4. Auto-compute: logarithmic formula, then `MIN(auto_degree, parallelism)`.

## Threading Model

- A **single named thread pool** (`"parallel-query"`) is owned by `worker_manager_global` (singleton). Capacity = `PRM_ID_MAX_PARALLEL_WORKERS`.
- Each parallel query **reserves** N slots from the pool via an atomic CAS loop (non-blocking). Returns 0 on contention.
- Tasks are `cubthread::entry_task` subclasses dispatched through `worker_manager_global::push_task()` → `cubthread::get_manager()->push_task(pool, task)`.
- Main thread **spins** on `m_active_tasks` in `wait_workers()` until all worker tasks call `pop_task()` (via `retire()`).

> [!warning] Spin wait in wait_workers
> `worker_manager::wait_workers()` uses `std::this_thread::yield()` in a busy-spin. This is intentional for short-lived parallel bursts but means the main thread is not sleeping. Avoid very high parallelism on systems with fewer physical cores.

## Interrupt Protocol

`px_interrupt.hpp` defines three orthogonal types:

| Class | Role |
|-------|------|
| `interrupt` | Atomic interrupt code (enum): `NO_INTERRUPT`, `USER_INTERRUPTED_FROM_MAIN_THREAD`, `USER_INTERRUPTED_FROM_WORKER_THREAD`, `ERROR_*`, `INST_NUM_SATISFIED`, `JOB_ENDED` |
| `atomic_instnum` | Atomic tuple counter for `ROWNUM`/`INST_NUM` early-exit |
| `err_messages_with_lock` | Mutex-protected vector of `cuberr::er_message*` — workers snapshot their thread-local error and push it here; main thread replays on error |

> [!key-insight] Error propagation across threads
> CUBRID uses thread-local error context (`cuberr::context`). Workers must explicitly move their error into `err_messages_with_lock` before exiting, otherwise the main thread cannot see it. `move_top_error_message_to_this()` swaps the worker's current error level into the shared list.

## Build Mode Guard

All files except `px_parallel.hpp/cpp` are guarded by:
```cpp
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Wrong module
#endif
```
`px_parallel.cpp` itself only reads system parameters and thread counts — usable in any build mode.

## Sub-Components

- [[components/parallel-worker-manager|parallel-worker-manager]] — pool lifecycle and reservation
- [[components/parallel-task-queue|parallel-task-queue]] — thread-safe MPMC queue and callable_task wrapper
- [[components/parallel-hash-join|parallel-hash-join]] — hash join parallelism
- [[components/parallel-heap-scan|parallel-heap-scan]] — heap scan parallelism
- [[components/parallel-query-execute|parallel-query-execute]] — subquery parallelism
- [[components/parallel-sort|parallel-sort]] — external sort parallelism

## Integration with Query Execution

The parallel subsystem hooks into the existing [[Query Processing Pipeline]] at the XASL executor level (`src/query/query_executor.c`). Non-parallel paths remain unchanged; the executor selects the parallel path when `compute_parallel_degree()` returns > 0 and `worker_manager::try_reserve_workers()` succeeds.

## Related

- Parent: [[modules/src|src]]
- [[components/storage|storage]] (heap files, page buffer accessed by parallel heap scan)
- [[Query Processing Pipeline]]
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
