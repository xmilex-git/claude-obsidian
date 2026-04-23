---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/"
status: developing
purpose: "Worker pool lifecycle, per-query worker reservation, and global singleton manager for parallel query execution"
key_files:
  - "px_worker_manager.hpp (per-query worker_manager class)"
  - "px_worker_manager.cpp (try_reserve_workers, push_task, wait_workers, release_workers)"
  - "px_worker_manager_global.hpp (singleton worker_manager_global)"
  - "px_worker_manager_global.cpp (pool init/destroy, atomic CAS reservation)"
public_api:
  - "worker_manager::try_reserve_workers(num_workers) -> worker_manager*"
  - "worker_manager::push_task(entry_task*)"
  - "worker_manager::wait_workers()"
  - "worker_manager::release_workers()"
  - "worker_manager::pop_task() (called by task on retire)"
  - "worker_manager_global::get_manager() (Meyer's singleton)"
  - "worker_manager_global::init() / destroy()"
tags:
  - component
  - cubrid
  - parallel
  - query
related:
  - "[[modules/src|src]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_worker_manager` — Parallel Worker Pool Management

Two-layer design: a **per-query** `worker_manager` (light, stack-friendly reservation handle) and a **process-global** `worker_manager_global` singleton that owns the actual thread pool.

## Layer 1 — `worker_manager_global` (Singleton)

Defined in `px_worker_manager_global.hpp`. Meyer's singleton via `get_manager()`.

```
worker_manager_global
  ├── m_worker_pool : cubthread::worker_pool_type*   (named "parallel-query")
  ├── m_capacity    : int                            (= PRM_ID_MAX_PARALLEL_WORKERS)
  ├── m_available   : std::atomic<int>               (available slot count)
  └── m_init_flag   : std::once_flag
```

### Lifecycle

| Method | Called by | Behaviour |
|--------|-----------|-----------|
| `init()` | Server startup | Calls `thread_create_worker_pool(pool_size, 1, "parallel-query", ...)` once. No-op if `max_parallel_workers < 2`. |
| `destroy()` | Server shutdown | Asserts all workers returned (`m_available == m_capacity`), then destroys pool. |

### Worker Reservation (CAS loop)

`try_reserve_workers(num_workers)` atomically subtracts from `m_available`:
```
while (true) {
  available = m_available.load()
  if (available < min_degree) return 0      // not enough — fallback to serial
  reserved = min(num_workers, available)
  if (CAS(m_available, available, available - reserved)) return reserved
  yield()  // retry on CAS failure
}
```
Minimum degree: 2 for heap/hash/sort; 1 for subquery.

## Layer 2 — `worker_manager` (Per-Query)

Allocated on the thread heap via `db_private_alloc` (not stack — size is known, but construction must use `placement_new`).

```
worker_manager
  ├── m_active_tasks    : std::atomic<int>   (tasks in flight)
  └── m_reserved_workers: int                (slots borrowed from global)
```

### Typical Usage Pattern

```cpp
// 1. Reserve
worker_manager *wm = worker_manager::try_reserve_workers(degree);
if (wm == nullptr) { /* fall back to serial */ }

// 2. Dispatch tasks
for (int i = 0; i < degree; i++) {
    auto *task = new callable_task(wm, lambda);
    wm->push_task(task);
}

// 3. Wait (spin-yield)
wm->wait_workers();

// 4. Release (returns slots, frees wm via db_private_free)
wm->release_workers();
```

> [!key-insight] Tasks self-decrement via pop_task
> Each `callable_task::retire()` calls `worker_manager::pop_task()` which does `m_active_tasks.fetch_sub(1)`. `wait_workers()` spins until `m_active_tasks == 0`. The worker manager does NOT use condition variables — it relies on the CUBRID worker pool's own scheduling to prevent starvation.

> [!warning] release_workers calls wait_workers internally
> Calling `release_workers()` always calls `wait_workers()` first, so double-waiting is safe but redundant. The destructor also calls `release_workers()`, providing RAII-like cleanup.

## Pool Registration

`px_worker_manager_global.cpp` uses a `REGISTER_WORKERPOOL` macro:
```cpp
REGISTER_WORKERPOOL(parallel_query, []() {
    return prm_get_integer_value(PRM_ID_MAX_PARALLEL_WORKERS);
});
```
This registers the pool capacity with CUBRID's thread manager infrastructure.

## Task Queue Size

The constant `TASK_QUEUE_SIZE_PER_CORE = 2` exists in `worker_manager_global` but the actual task queue is managed by `cubthread::worker_pool_type` — each worker processes tasks from the shared pool queue.

## Related

- [[components/parallel-query|parallel-query]] — parent overview
- [[components/parallel-task-queue|parallel-task-queue]] — callable_task and thread_safe_queue
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
