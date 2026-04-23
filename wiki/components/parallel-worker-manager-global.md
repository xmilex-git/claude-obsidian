---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/"
status: active
purpose: "Process-singleton worker pool for all parallel queries: lazy init via std::call_once, atomic slot accounting, named cubthread worker pool"
key_files:
  - "px_worker_manager_global.hpp (worker_manager_global class — Meyer's singleton)"
  - "px_worker_manager_global.cpp (init, destroy, try_reserve_workers, release_workers, push_task)"
public_api:
  - "worker_manager_global::get_manager() -> worker_manager_global&  (Meyer's singleton)"
  - "worker_manager_global::init()         (server startup)"
  - "worker_manager_global::destroy()      (server shutdown)"
  - "worker_manager_global::try_reserve_workers(int) -> int  (friend of worker_manager)"
  - "worker_manager_global::release_workers(int)             (friend of worker_manager)"
  - "worker_manager_global::push_task(entry_task*)           (friend of worker_manager)"
tags:
  - component
  - cubrid
  - parallel
  - query
  - singleton
  - thread-pool
related:
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[components/worker-pool|worker-pool]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_worker_manager_global` — Singleton Global Worker Pool

`worker_manager_global` is the process-wide singleton that owns the single named `cubthread::worker_pool_type` used by all parallel queries. It tracks available worker slots via an atomic counter and dispatches tasks to the underlying thread pool.

For the per-query handle that borrows slots from this pool, see [[components/parallel-worker-manager|parallel-worker-manager]].

---

## Class / Function Inventory

### `worker_manager_global` (in `px_worker_manager_global.hpp` / `.cpp`)

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `get_manager` | static inline | `static worker_manager_global& get_manager()` | Meyer's singleton — returns the one instance |
| `init` | method | `void init()` | `std::call_once` — creates the `"parallel-query"` worker pool |
| `destroy` | method | `void destroy()` | Asserts all workers returned; destroys pool via `cubthread::get_manager()` |
| `try_reserve_workers` | private | `int try_reserve_workers(const int num_workers)` | CAS loop on `m_available`; returns how many were reserved (0 = fail) |
| `release_workers` | private | `void release_workers(const int num_workers)` | `m_available.fetch_add(num_workers)` |
| `push_task` | private | `void push_task(cubthread::entry_task*)` | `cubthread::get_manager()->push_task(m_worker_pool, task)` |

### Member Variables

| Field | Type | Initial value | Role |
|-------|------|---------------|------|
| `m_worker_pool` | `cubthread::worker_pool_type*` | `nullptr` | The named thread pool; null until `init()` succeeds |
| `m_init_flag` | `std::once_flag` | default | Guards `std::call_once` in `init()` |
| `m_available` | `std::atomic<int>` | `0` | Available worker slots; set to `m_capacity` after successful init |
| `m_capacity` | `int` | `0` | Total worker capacity = `PRM_ID_MAX_PARALLEL_WORKERS` |

### Private Constant

| Constant | Value | Role |
|----------|-------|------|
| `TASK_QUEUE_SIZE_PER_CORE` | `2` | Defined but used by the pool's per-core task queue sizing (passed as the second arg to `thread_create_worker_pool`) |

---

## Meyer's Singleton Pattern

```cpp
static worker_manager_global& get_manager() {
    static worker_manager_global instance;
    return instance;
}
```

The singleton is a function-local static. In C++11 and later, initialization of function-local statics is guaranteed to be thread-safe (the compiler emits a guard). Destruction occurs at program exit via normal static destructor rules — `~worker_manager_global()` calls `destroy()`.

The constructor zeroes all fields (`m_worker_pool = nullptr`, `m_available = 0`, `m_capacity = 0`). The actual pool creation is deferred to `init()`.

---

## Initialization — `std::call_once`

```cpp
void worker_manager_global::init() {
    std::call_once(m_init_flag, [this]() {
        int max_parallel_workers = prm_get_integer_value(PRM_ID_MAX_PARALLEL_WORKERS);

        if (max_parallel_workers < 2) {
            // parallel execution requires at least 2 workers; disable
            return;
        }

        int pool_size = max_parallel_workers;
        m_worker_pool = thread_create_worker_pool(
            pool_size,                    // total worker count
            1,                            // task queue size per core = TASK_QUEUE_SIZE_PER_CORE? (hardcoded 1 here)
            "parallel-query",             // pool name (visible in thread manager registry)
            thread_get_entry_manager()    // entry manager for pool worker threads
        );

        if (m_worker_pool == nullptr) return;  // pool creation failed

        m_capacity  = max_parallel_workers;
        m_available = max_parallel_workers;    // all slots available
    });
}
```

> [!key-insight] `std::call_once` for lazy pool creation
> The pool is created inside `init()` which is called once during server startup (not at first query). `std::call_once` with `m_init_flag` ensures the pool is created exactly once even if multiple threads call `init()` concurrently (e.g. during rapid query startup). After `init()` returns, subsequent calls to `init()` are no-ops — the `once_flag` is permanently "done."
>
> The `min < 2` guard means that on a system where `PRM_ID_MAX_PARALLEL_WORKERS` is set to 0 or 1, no pool is created and `m_worker_pool` stays `nullptr`. All reservation calls then see `m_available = 0` and return 0 immediately.

> [!warning] `init()` called only once; parameter changes require restart
> `m_capacity` and `m_available` are set from `PRM_ID_MAX_PARALLEL_WORKERS` inside the `call_once` lambda. Like `compute_parallel_degree`'s parameter caching, any runtime change to `PRM_ID_MAX_PARALLEL_WORKERS` takes no effect until the next server restart.

---

## Pool Registration Macro

```cpp
REGISTER_WORKERPOOL(parallel_query, []() {
    return prm_get_integer_value(PRM_ID_MAX_PARALLEL_WORKERS);
});
```

This macro (from `thread_manager.hpp`) registers the `"parallel-query"` pool capacity with CUBRID's thread manager infrastructure at static-init time. The lambda is called by the thread manager during capacity interrogation. It is separate from `init()` — registration happens at static init, but the actual pool object is created on demand inside `init()`.

---

## Reservation — `try_reserve_workers`

```cpp
int worker_manager_global::try_reserve_workers(const int num_workers) {
    assert(num_workers > 0);
    assert(num_workers <= PRM_MAX_PARALLELISM);

    int requested = MIN(num_workers, PRM_MAX_PARALLELISM);

    // minimum degree: 2 for scan/join/sort, 1 for subquery
    const int min_degree = (requested == 1) ? 1 : 2;

    int available = m_available.load();    // non-atomic read as starting estimate

    while (true) {
        if (available < min_degree)
            return 0;                      // not enough workers → serial fallback

        int reserved = (requested <= available) ? requested : available;

        if (m_available.compare_exchange_weak(available, available - reserved))
            return reserved;              // CAS succeeded → we own these slots

        // CAS failed: available was updated with actual current value; retry
        std::this_thread::yield();
    }
}
```

Key semantics:
- **Partial grant**: if fewer than `requested` workers are available but `>= min_degree`, a smaller number is granted (`reserved = available`). The caller (per-query `worker_manager`) stores exactly how many were granted.
- **Full fail**: returns 0 when `available < min_degree`. The per-query manager returns `nullptr` and the query falls back to serial execution.
- **CAS failure handling**: `compare_exchange_weak` updates `available` on failure with the actual current value (the `expected` parameter is passed by reference). On retry, `available` is already fresh — no extra load needed.
- **Yield on contention**: after a CAS failure, `std::this_thread::yield()` gives other threads a chance to complete their own reservations. Contention is expected to be low (reservation happens once per parallel query, not per row).

> [!key-insight] Partial grant semantics are correct
> The caller may request `N` workers but receive `M < N`. The per-query `worker_manager` stores `m_reserved_workers = M` and dispatches at most `M` tasks. The sort and scan subsystems use `get_reserved_workers()` to know the actual degree. This means a partially-granted query runs at lower parallelism rather than being rejected entirely — better throughput under memory pressure.

---

## Release — `release_workers`

```cpp
void worker_manager_global::release_workers(const int num_workers) {
    assert(num_workers > 0);
    assert(m_worker_pool != nullptr);
    assert(m_available.load() + num_workers <= m_capacity);  // no over-release

    m_available.fetch_add(num_workers);   // simple atomic add; no CAS needed
}
```

Release is a simple `fetch_add` — no CAS required because there is no constraint on the order in which workers are released. The assertion guards against a programming error where more slots are returned than were ever allocated.

---

## Task Dispatch — `push_task`

```cpp
void worker_manager_global::push_task(cubthread::entry_task *task) {
    assert(task != nullptr);
    assert(m_worker_pool != nullptr);
    cubthread::get_manager()->push_task(m_worker_pool, task);
}
```

Delegates directly to `cubthread::manager::push_task`. The thread manager enqueues the task in the `"parallel-query"` pool's work queue. A free worker picks it up and calls `task->execute(entry)`, then `task->retire()`.

---

## Destroy — `destroy`

```cpp
void worker_manager_global::destroy() {
    if (m_worker_pool == nullptr) return;  // init was not called or failed

    assert(m_available.load() == m_capacity);  // all workers must be returned first

    cubthread::get_manager()->destroy_worker_pool(m_worker_pool);
    m_worker_pool = nullptr;
}
```

The `m_available == m_capacity` assertion ensures that no queries are still holding reservations when the server shuts down. Violating this assertion indicates a missing `release_workers()` call.

---

## Interaction with [[components/thread-manager|thread-manager]]

```
worker_manager_global::init()
  └─ thread_create_worker_pool(pool_size, 1, "parallel-query", entry_mgr)
        └─ cubthread::manager::create_worker_pool(...)
              └─ registers pool in manager's pool registry
              └─ creates pool_size worker threads

worker_manager_global::push_task(task)
  └─ cubthread::manager::push_task(m_worker_pool, task)
        └─ pool selects an idle worker
        └─ worker calls task->execute(entry)
        └─ worker calls task->retire()

worker_manager_global::destroy()
  └─ cubthread::manager::destroy_worker_pool(m_worker_pool)
        └─ signals all workers to stop
        └─ joins worker threads
        └─ deregisters pool from manager registry
```

---

## Execution Path — Reservation to Release

```
Server startup
  └─ worker_manager_global::init()
        std::call_once → thread_create_worker_pool("parallel-query", N)
        m_capacity = m_available = N

Per-query (multiple concurrent queries possible)
  Query A: try_reserve_workers(4) → CAS m_available: N → N-4 → returns 4
  Query B: try_reserve_workers(4) → CAS m_available: N-4 → N-8 → returns 4
  Query C: try_reserve_workers(8) →
    available = N-8, requested = 8, reserved = min(8, N-8)
    → partial grant or 0 if N-8 < 2

  [tasks execute in pool workers]

  Query A: release_workers(4) → fetch_add m_available: N-8 → N-4
  Query B: release_workers(4) → fetch_add m_available: N-4 → N

Server shutdown
  └─ worker_manager_global::destroy()
        assert m_available == N
        cubthread::manager::destroy_worker_pool(m_worker_pool)
```

---

## Lifecycle

| Stage | Method | Notes |
|-------|--------|-------|
| Static init | `get_manager()` first call | Meyer's singleton constructed; fields zeroed |
| Server start | `init()` | `std::call_once` creates pool; sets capacity and available |
| Per-query reserve | `try_reserve_workers(N)` | CAS loop; partial grant possible |
| Per-query dispatch | `push_task(task)` | Delegates to `cubthread::manager` |
| Per-query release | `release_workers(N)` | `fetch_add(N)` to `m_available` |
| Server stop | `destroy()` | Asserts all returned; destroys pool |
| Static destruction | `~worker_manager_global()` | Calls `destroy()` again (safe — no-op if `m_worker_pool == nullptr`) |

---

## Constraints

- **Build mode**: `SERVER_MODE` or `SA_MODE` only.
- **`init()` must be called before any reservation** — if `m_worker_pool == nullptr`, `release_workers` and `push_task` will assert-fail.
- **`destroy()` requires all workers released** — `m_available == m_capacity` is asserted.
- **Copy and assignment deleted** — only the Meyer's singleton instance is valid.
- **`PRM_ID_MAX_PARALLEL_WORKERS < 2`** results in no pool being created; `m_available` stays 0; all reservations return 0.
- **No try-again mechanism** — `init()` uses `call_once`: if the pool creation fails (`m_worker_pool = nullptr`), the `once_flag` is still set and `init()` will never be called again.

> [!warning] Pool creation failure is permanent
> If `thread_create_worker_pool` returns `nullptr` (e.g. OS thread limit exceeded), the `once_flag` is still consumed and no retry is possible without restarting the server. All subsequent reservation calls return 0 (serial fallback). No error is propagated to the caller.

---

## Related

- [[components/parallel-query|parallel-query]] — degree selection that feeds reservation requests
- [[components/parallel-worker-manager|parallel-worker-manager]] — per-query handle that calls these methods
- [[components/thread-manager|thread-manager]] — `cubthread::manager` that owns the pool registry
- [[components/worker-pool|worker-pool]] — `cubthread::worker_pool_type` details
- [[components/system-parameter|system-parameter]] — `PRM_ID_MAX_PARALLEL_WORKERS`, `PRM_MAX_PARALLELISM`
- [[Build Modes (SERVER SA CS)]]
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
