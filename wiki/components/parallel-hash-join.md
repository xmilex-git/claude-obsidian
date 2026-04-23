---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_hash_join/"
status: active
purpose: "Parallel build and probe phases of hash join using a shared task_manager and per-task XASL execution context"
key_files:
  - "px_hash_join.hpp — public API: build_partitions, execute_partitions"
  - "px_hash_join.cpp — top-level dispatch: outer/inner split phases then join phase"
  - "px_hash_join_spawn_manager.{hpp,cpp} — per-worker TLS XASL structure spawner"
  - "px_hash_join_task_manager.{hpp,cpp} — task_manager, task_execution_guard, split_task, join_task"
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
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/xasl|xasl]]"
  - "[[components/btree|btree]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_hash_join/` — Parallel Hash Join (Hub)

Parallelises the build and probe phases of `HASHJOIN_MANAGER` using worker threads. Three files form a strict pipeline: `px_hash_join` (entry/dispatch) → `px_hash_join_task_manager` (per-task execution and coordination) → `px_hash_join_spawn_manager` (per-worker TLS XASL spawning for join phase).

## Sub-pages

| Page | Content |
|------|---------|
| [[components/parallel-hash-join-spawn-manager]] | `spawn_manager`: per-worker TLS XASL structure spawner |
| [[components/parallel-hash-join-task-manager]] | `task_manager`, `task_execution_guard`, `split_task`, `join_task` |

## Three-File Pipeline

```
px_hash_join.cpp
  build_partitions()          execute_partitions()
       |                              |
       v                              v
  px_hash_join_task_manager.cpp
  task_manager::push_task()    task_manager::push_task()
  task_manager::join()         task_manager::join()
       |                              |
       v                              v
  [split_task workers]         [join_task workers]
       |                              |
       v                              v
                 px_hash_join_spawn_manager.cpp
                 spawn_manager::get_instance(thread_ref)
                 spawn_manager::get_val_descr / get_*_regu_list_pred
```

## Execution Path

```
caller
  │
  ├─ build_partitions(thread_ref, manager, split_info)
  │    ├─ hjoin_init_shared_split_info()
  │    ├─ Phase A — outer split:
  │    │    for i in [0, num_parallel_threads):
  │    │        new split_task(outer, shared_info, i)
  │    │    task_manager.join()          ← CV-wait
  │    ├─ (check error + clear shared_info)
  │    └─ Phase B — inner split:
  │         for i in [0, num_parallel_threads):
  │             new split_task(inner, shared_info, i)
  │         task_manager.join()
  │
  └─ execute_partitions(thread_ref, manager)
       ├─ Phase C — join:
       │    for i in [0, num_parallel_threads):
       │        new join_task(contexts, shared_info, i)
       │    task_manager.join()
       └─ merge: for each context, hjoin_merge_qlist() into single_context.list_id
```

> [!key-insight] Two independent split phases then one join phase
> `build_partitions` runs two sequential barrier-synchronised passes (outer, inner) before `execute_partitions` runs join tasks. Each phase allocates a fresh `task_manager` on the stack and calls `join()` before proceeding — there is no overlap between phases.

## Constraints

- **Build mode**: active in `SERVER_MODE` and `SA_MODE` only (worker pool exists in both).
- **Memory**: `split_task` and `join_task` are heap-allocated with `new`; they self-delete in `base_task::retire()`.
- **Threading**: `HASHJOIN_SHARED_SPLIT_INFO::scan_mutex` and `part_mutexes[]` protect shared page cursor and per-partition list files. `HASHJOIN_SHARED_JOIN_INFO::scan_mutex` protects the context index.
- **Interrupt**: `task_manager::check_interrupt` polls `logtb_is_interrupted_tran` per page/context iteration. On interrupt, all workers signal via `m_has_error` + `notify_stop()`.
- **XASL spawning**: `spawn_manager` instances are TLS singletons; each worker calls `destroy_instance()` at the end of `join_task::execute()`.

## Lifecycle

```
1. caller allocates HASHJOIN_MANAGER with worker_manager already reserved
2. build_partitions():
      stack-allocates task_manager
      dispatches split_tasks (outer then inner) via worker_manager
      join() waits for all splits to complete
3. execute_partitions():
      stack-allocates fresh task_manager
      dispatches join_tasks
      join() waits
      merges per-context list_ids into single_context
4. HASHJOIN_MANAGER destroyed by caller (not by this layer)
```

## Related

- [[components/parallel-hash-join-spawn-manager]] — XASL structure cloning per worker thread
- [[components/parallel-hash-join-task-manager]] — `split_task` partition logic, `join_task` build+probe
- [[components/parallel-query|parallel-query]] — degree selection and pool management
- [[components/parallel-worker-manager|parallel-worker-manager]] — task dispatch
- [[components/xasl|xasl]] — XASL tree structures used by join tasks
- [[components/btree|btree]] — index probe may occur inside `hjoin_execute`
