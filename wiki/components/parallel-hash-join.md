---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_hash_join/"
status: developing
purpose: "Parallel build and probe phases of hash join using a shared task_manager and per-task XASL execution context"
key_files:
  - "px_hash_join.hpp (public API: build_partitions, execute_partitions)"
  - "px_hash_join.cpp (dispatch loop: split_task and join_task to worker pool)"
  - "px_hash_join_task_manager.hpp (task_manager, base_task, split_task, join_task, task_execution_guard)"
public_api:
  - "parallel_query::hash_join::build_partitions(thread_ref, manager, split_info) -> int"
  - "parallel_query::hash_join::execute_partitions(thread_ref, manager) -> int"
tags:
  - component
  - cubrid
  - parallel
  - query
  - hash-join
related:
  - "[[modules/src|src]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_hash_join/` — Parallel Hash Join

Implements two-phase parallelism over `HASHJOIN_MANAGER`: first split/partition input pages in parallel (`build_partitions`), then execute build + probe across partitions in parallel (`execute_partitions`). Both phases use the same task dispatch pattern.

## Two-Phase Protocol

```
build_partitions(thread_ref, manager, split_info)
  │
  ├─ Phase 1: outer split
  │    for task_index in [0, num_parallel_threads):
  │        push split_task(outer, shared_info, task_index)
  │    task_manager.join()
  │
  └─ Phase 2: inner split
       for task_index in [0, num_parallel_threads):
           push split_task(inner, shared_info, task_index)
       task_manager.join()

execute_partitions(thread_ref, manager)
  │
  ├─ for task_index in [0, num_parallel_threads):
  │      push join_task(contexts, shared_info, task_index)
  │  task_manager.join()
  │
  └─ merge per-context list_ids into manager->single_context.list_id
```

## `task_manager`

A local (stack-allocated) coordinator within each call to `build_partitions` or `execute_partitions`. Not the same as the global `parallel_query::worker_manager`.

```
task_manager
  ├── m_worker_manager      : parallel_query::worker_manager*
  ├── m_main_thread_ref     : cubthread::entry&
  ├── m_main_error_context  : cuberr::context&
  ├── m_active_tasks        : int
  ├── m_active_tasks_mutex  : std::mutex
  ├── m_all_tasks_done_cv   : std::condition_variable
  └── m_has_error           : std::atomic<bool>
```

`task_manager::join()` **condition-variable waits** (not spin) until `m_active_tasks == 0`. This differs from `worker_manager::wait_workers()` which busy-spins.

> [!key-insight] Hash join uses CV-wait, not spin-wait
> Unlike the base `worker_manager::wait_workers()` (spin-yield), the hash join's `task_manager::join()` uses a `condition_variable`. This is appropriate because hash join tasks may take longer (building hash tables, probing) than short heap scan iterations.

## `task_execution_guard` — Worker Thread Identity Borrowing

When a worker thread executes a task, it must appear to be operating within the originating transaction:

```cpp
task_execution_guard(thread_ref, task_manager) {
    thread_ref.m_px_orig_thread_entry = &main_thread_ref;
    thread_ref.conn_entry  = main_thread_ref.conn_entry;
    thread_ref.tran_index  = main_thread_ref.tran_index;
    thread_ref.on_trace    = main_thread_ref.on_trace;
    thread_ref.push_resource_tracks();
}
~task_execution_guard() {
    thread_ref.conn_entry = nullptr;
    thread_ref.pop_resource_tracks();
}
```

> [!key-insight] Worker threads borrow the main thread's transaction context
> Each worker copies `conn_entry` and `tran_index` from the main thread. This is how CUBRID makes parallel workers "join" the calling transaction's MVCC snapshot and lock context. The `m_px_orig_thread_entry` field marks the thread as a parallel worker.

## Task Hierarchy

```
cubthread::entry_task
  └── base_task (holds task_manager&, HASHJOIN_MANAGER*, index)
        ├── split_task  (HASHJOIN_INPUT_SPLIT_INFO*, HASHJOIN_SHARED_SPLIT_INFO*)
        │     execute(): get_next_page() in loop, partition pages into split buckets
        └── join_task   (HASHJOIN_CONTEXT*, HASHJOIN_SHARED_JOIN_INFO*)
              execute(): get_next_context(), build hash table + probe
```

## Error Handling

On error in any task:
1. Task sets `task_manager::m_has_error = true`.
2. `task_manager::notify_stop()` signals other tasks via interrupt.
3. After `join()`, caller checks `has_error()` → calls `clear_interrupt(thread_ref)` which replays the worker's error into the main thread's error context.

## Tracing

Both phases are trace-aware (`thread_is_on_trace`). Trace stats are aggregated per-worker and drained into the manager's `HASHJOIN_STATS` via `hjoin_trace_drain_worker_stats`.

## Related

- [[components/parallel-query|parallel-query]] — degree selection and pool management
- [[components/parallel-worker-manager|parallel-worker-manager]] — task dispatch
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
