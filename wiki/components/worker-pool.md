---
type: component
parent_module: "[[modules/src|src]]"
path: "src/thread/"
status: stable
purpose: "Generic thread pool with core-partitioned task queues and optional statistics collection; dispatches entry_task instances to pooled threads"
key_files:
  - "thread_worker_pool.hpp (abstract worker_pool, core, worker)"
  - "thread_worker_pool_impl.hpp (worker_pool_impl<Stats> — concrete)"
  - "thread_manager.hpp (worker_pool_type and stats_worker_pool_type aliases)"
public_api:
  - "worker_pool::execute(task_type*) — enqueue task, guaranteed delivery"
  - "worker_pool::execute_on_core(task_type*, core_hash, is_temp) — affinity dispatch"
  - "worker_pool::warmup() — pre-spin all worker threads"
  - "worker_pool::stop_execution() — drain and join all threads"
  - "worker_pool::get_name() — pool name string"
  - "worker_pool::get_worker_count() / get_core_count()"
  - "worker_pool_impl<Stats>::map_running_contexts(func, args...) — iterate live contexts"
  - "worker_pool_impl<Stats>::map_cores(func, args...)"
  - "thread_create_worker_pool(pool_size, core_count, name, entry_mgr, pool_threads)"
  - "thread_create_stats_worker_pool(pool_size, core_count, name, entry_mgr, ...)"
tags:
  - component
  - cubrid
  - thread
  - concurrency
related:
  - "[[components/thread|thread]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[components/entry-task|entry-task]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/vacuum|vacuum]]"
  - "[[components/lockfree|lockfree]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cubthread::worker_pool` — Worker Pool

The worker pool is a core primitive of the `cubthread` namespace. It provides a reusable thread pool where tasks (`entry_task` instances) are dispatched to threads that carry a `THREAD_ENTRY` context.

## Class Hierarchy

```
cubthread::worker_pool          (abstract base — thread_worker_pool.hpp)
  └── worker_pool_impl<Stats>   (concrete — thread_worker_pool_impl.hpp)
        └── core_impl[]         (per-core partition — manages worker threads)
              └── worker[]      (std::thread wrapper; pulls tasks from queue)
```

`worker_pool::core` and `worker_pool::core::worker` are also abstract nested classes with virtual interfaces, allowing alternative implementations.

## Template Parameter `Stats`

`worker_pool_impl<Stats>` is a bool template:

| Value | Alias | Use |
|-------|-------|-----|
| `false` | `worker_pool_type` | Standard pools (parallel-query, vacuum workers, connection handlers) |
| `true` | `stats_worker_pool_type` | Pools with per-operation latency / queue-depth statistics |

The stats variant adds a `stats_base` specialization that accumulates `cubperf::stat_value` counters per worker per core.

## Core Partitioning

A pool can be split into multiple **cores** (not necessarily aligned to physical CPU cores, but the concept follows NUMA-locality thinking):

```
worker_pool_impl
  core_impl[0]  → worker[0], worker[1], ...   (task queue Q0)
  core_impl[1]  → worker[2], worker[3], ...   (task queue Q1)
  ...
```

- `execute(task)` — dispatches to the next core by **round-robin** (`m_round_robin_counter`)
- `execute_on_core(task, core_hash, is_temp)` — dispatches to `core_hash % core_count`; used for affinity-sensitive work (e.g., method-mode execution where a specific transaction context is pinned to a core)

## Thread Lifecycle (Low-Load vs High-Load)

```
Low-load  (tasks < threads):
  worker finishes task → no new task in queue → thread retires (returns entry, sleeps)
  new task arrives → wake idle thread or spawn new thread (up to pool_size)

High-load (tasks >= threads):
  worker finishes task → pulls next from queue → continues with same THREAD_ENTRY
  context is shared between sequential tasks on the same worker
```

`pool_threads = true` forces all worker threads to stay alive permanently (used in `PRM_ID_PERF_TEST_MODE`).

`idle_timeout` (default 5 s for stats pools) controls how long an idle thread waits before exiting.

## Construction

Pools are created via `cubthread::manager::create_worker_pool<Res>`:

```cpp
worker_pool_type* pool = cubthread::get_manager()->create_worker_pool<worker_pool_type>(
    pool_size,    // total worker threads
    core_count,   // number of cores (partitions)
    "my-pool",    // name (used as OS thread name prefix)
    entry_mgr,    // entry_manager for context lifecycle
    pool_threads  // keep alive?
);
```

Or via the C shim:

```cpp
worker_pool_type* pool = thread_create_worker_pool(pool_size, core_count, name, entry_mgr);
```

The manager checks that `m_available_entries_count >= pool_size`; if not, returns NULL.

## Stopping / Draining

`stop_execution()` (called by `manager::destroy_worker_pool`):
1. Sets `m_stopped = true`
2. Signals all workers
3. Joins all threads
4. Tasks still in queue are **discarded** (not executed)

## Integration with Parallel Query

[[components/parallel-query|parallel-query]] uses a single named pool `"parallel-query"` with `core_count=1`:

```cpp
// in px_worker_manager_global.cpp
REGISTER_WORKERPOOL(parallel_query, []() { return prm_get_integer_value(PRM_ID_MAX_PARALLEL_WORKERS); });
// ...
m_worker_pool = thread_create_worker_pool(m_capacity, 1, "parallel-query", entry_mgr);
```

[[components/parallel-worker-manager|parallel-worker-manager]] then wraps this pool with per-query atomic slot reservation on top. Tasks are `cubthread::entry_task` subclasses; each calls `worker_manager::pop_task()` from its `retire()`.

> [!key-insight] Pool is SERVER_MODE only
> In SA_MODE, `create_worker_pool` returns NULL and `push_task(NULL, task)` executes the task inline. All pool-dependent subsystems (parallel query, vacuum workers, etc.) must handle the NULL pool case.

## Statistics

`stats_worker_pool_type` (`worker_pool_impl<true>`) adds per-core performance counters retrievable via `get_stats(stat_value* out)` and loggable with `er_log_stats()`.

## Related

- [[components/thread|thread]] — parent namespace overview
- [[components/thread-manager|thread-manager]] — pool creation, registration, and destruction
- [[components/entry-task|entry-task]] — the task type dispatched through pools
- [[components/parallel-worker-manager|parallel-worker-manager]] — layer above this pool for parallel query
- [[components/vacuum|vacuum]] — vacuum worker pool consumer
- Source: [[sources/cubrid-src-thread|cubrid-src-thread]]
