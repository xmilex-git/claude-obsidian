---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/"
status: active
purpose: "Per-query worker reservation handle: reserves slots from the global pool, tracks in-flight tasks, spins until all tasks complete"
key_files:
  - "px_worker_manager.hpp (worker_manager class)"
  - "px_worker_manager.cpp (try_reserve_workers, push_task, wait_workers, release_workers)"
public_api:
  - "worker_manager::try_reserve_workers(num_workers) -> worker_manager* or nullptr"
  - "worker_manager::push_task(entry_task*)"
  - "worker_manager::wait_workers()"
  - "worker_manager::release_workers()"
  - "worker_manager::pop_task()  (called by callable_task::retire)"
  - "worker_manager::get_reserved_workers() -> int"
tags:
  - component
  - cubrid
  - parallel
  - query
related:
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager-global|parallel-worker-manager-global]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/entry-task|entry-task]]"
  - "[[components/lockfree|lockfree]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_worker_manager` — Per-Query Worker Manager

`worker_manager` is a lightweight, per-query reservation handle. It does NOT own the thread pool — it merely borrows N slots from the process-global `worker_manager_global` and tracks how many tasks are still in flight.

For the global singleton pool see [[components/parallel-worker-manager-global|parallel-worker-manager-global]].

---

## Class / Function Inventory

### `worker_manager` (in `px_worker_manager.hpp` / `.cpp`)

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `try_reserve_workers` | static method | `static worker_manager* try_reserve_workers(int num_workers)` | Allocates a `worker_manager` if the global pool has enough slots; returns `nullptr` on failure |
| `worker_manager()` | constructor | `worker_manager()` | Zeroes `m_active_tasks` and `m_reserved_workers` |
| `~worker_manager()` | destructor | `~worker_manager()` | Calls `release_workers()` — provides RAII-like cleanup |
| `release_workers` | method | `void release_workers()` | Calls `wait_workers()`, returns slots to global, destructs and frees `this` via `db_private_free` |
| `wait_workers` | method | `void wait_workers()` | Busy-spins with `std::this_thread::yield()` until `m_active_tasks == 0` |
| `push_task` | method | `void push_task(cubthread::entry_task*)` | Increments `m_active_tasks`, forwards task to global pool |
| `pop_task` | inline method | `void pop_task()` | `m_active_tasks.fetch_sub(1, release)` — called by `callable_task::retire()` |
| `get_reserved_workers` | inline const | `int get_reserved_workers() const` | Returns `m_reserved_workers` (how many slots were reserved) |

### Member variables

| Field | Type | Initial value | Role |
|-------|------|---------------|------|
| `m_active_tasks` | `std::atomic<int>` | `0` | Number of tasks currently executing in the pool |
| `m_reserved_workers` | `int` | `0` | Slots borrowed from `worker_manager_global::m_available` |

---

## Reservation Protocol — `try_reserve_workers`

```cpp
static worker_manager* try_reserve_workers(int num_workers) {
    // 1. CAS loop on global m_available (see parallel-worker-manager-global)
    int reserved = worker_manager_global::get_manager().try_reserve_workers(num_workers);
    if (reserved == 0) return nullptr;          // not enough workers → serial fallback

    // 2. Allocate on the calling thread's private heap
    THREAD_ENTRY *thread_p = thread_get_thread_entry_info();
    worker_manager *manager = (worker_manager *) db_private_alloc(thread_p, sizeof(worker_manager));
    if (manager == nullptr) {
        worker_manager_global::get_manager().release_workers(reserved);  // return what we took
        return nullptr;
    }

    // 3. Placement-new to properly construct atomics
    manager = placement_new(manager);
    manager->m_reserved_workers = reserved;
    return manager;
}
```

Key points:
- The object is heap-allocated via `db_private_alloc` (thread-local private heap), not stack-allocated, because `std::atomic<int>` needs a properly aligned, heap-stable address.
- `placement_new` calls the constructor on already-allocated memory, initializing `m_active_tasks` to 0.
- If allocation fails, the already-reserved global slots are returned immediately before returning `nullptr`.
- The global CAS loop may return fewer workers than requested (it takes `min(requested, available)` — see global page). The caller gets back exactly how many were reserved via `get_reserved_workers()`.

> [!key-insight] `try_reserve_workers` returns 0 on contention, not a partial grant
> When `m_available < min_degree` (2 for most types, 1 for subquery), the global CAS returns 0 and `try_reserve_workers` returns `nullptr`. There is no blocking wait — the caller immediately falls back to serial execution. This is the correct choice for a database query: adding a scheduling wait to the query critical path would be worse than running serially.

---

## Task Dispatch — `push_task`

```cpp
void worker_manager::push_task(cubthread::entry_task *task) {
    assert(task != nullptr);
    assert(m_reserved_workers > 0);
    assert(m_active_tasks.load() < m_reserved_workers);   // invariant: never exceed reservation
    m_active_tasks.fetch_add(1, std::memory_order_release);
    worker_manager_global::get_manager().push_task(task);  // → cubthread::get_manager()->push_task(pool, task)
}
```

The `memory_order_release` on the fetch_add pairs with the `memory_order_acquire` in `wait_workers`'s load, establishing the happens-before edge: all work done before `push_task` is visible to `wait_workers` when it observes the count drop back to 0.

---

## Completion Tracking — `wait_workers` and `pop_task`

### `wait_workers`

```cpp
void worker_manager::wait_workers() {
    assert(m_reserved_workers > 0);
    while (m_active_tasks.load(std::memory_order_acquire) > 0) {
        std::this_thread::yield();
    }
}
```

Pure busy-spin with `yield()`. No condition variable, no mutex.

> [!key-insight] Why busy-spin instead of condvar?
> `wait_workers` is a short-duration join point. Parallel sort, hash join, and heap scan all complete in microseconds to low milliseconds — the overhead of a condvar (kernel syscall, context switch, wakeup latency) would dominate. The busy-spin is the right tradeoff for this use case. By contrast, `SORT_WAIT_PARALLEL` uses `pthread_cond_wait` because the sort wait can span longer intervals and involves inter-process-style signaling from worker threads back to the main thread for per-partition status.

> [!warning] Busy-spin on low-core machines
> On a machine with 2 physical cores, `wait_workers` keeps the main thread's logical CPU burning. The guard `system_core_count <= 2 → return 0` in `compute_parallel_degree` is designed to prevent this — but if the guard is bypassed (e.g. a hint forces parallelism) the spin can degrade throughput.

### `pop_task` — called by `callable_task::retire()`

```cpp
// inline in header:
void pop_task() {
    m_active_tasks.fetch_sub(1, std::memory_order_release);
}
```

After `execute()` returns, CUBRID's worker pool calls `retire()` on the task. `callable_task::retire()` first calls `pop_task()` (decrement), nulls `m_worker_manager_p`, then calls `m_retire_f()` (which by default calls `delete this`). The ordering is critical: the counter must be decremented before `delete this`, so `wait_workers` never reads a counter that reflects tasks whose storage has already been freed.

---

## `release_workers`

```cpp
void worker_manager::release_workers() {
    if (m_reserved_workers == 0) return;        // already released

    wait_workers();                              // drain all in-flight tasks first

    worker_manager_global::get_manager().release_workers(m_reserved_workers);
    m_reserved_workers = 0;

    THREAD_ENTRY *thread_p = thread_get_thread_entry_info();
    this->~worker_manager();                    // explicit destructor
    db_private_free(thread_p, this);            // return memory to private heap
}
```

> [!warning] `release_workers` is not idempotent after first call
> After `release_workers()` sets `m_reserved_workers = 0` and frees `this`, no further calls are safe. The destructor calls `release_workers()` as a safety net, but if the caller also calls `release_workers()` explicitly, the first call frees the memory and the second call is a use-after-free. Callers must not call `release_workers()` and then let the object be destroyed.

---

## Execution Path — Sequence Diagram

```
Caller (main thread)                worker_manager_global       Pool worker thread
  │                                        │                         │
  ├─ try_reserve_workers(N) ─────────────► CAS m_available          │
  │   ◄──────────────────── worker_manager*                         │
  │                                        │                         │
  ├─ push_task(task1) ──── fetch_add(active,1) ──► push_task ──────► enqueue task1
  ├─ push_task(task2) ──── fetch_add(active,1) ──► push_task ──────► enqueue task2
  │                                        │                         │
  │                                        │         execute(task1) ─┤
  │                                        │         retire(task1) ──┼─ pop_task() → fetch_sub(active,1)
  │                                        │         execute(task2) ─┤
  │                                        │         retire(task2) ──┼─ pop_task() → fetch_sub(active,1)
  │                                        │                         │
  ├─ wait_workers() [spin on active==0]    │                         │
  │   ◄──── exits when active reaches 0   │                         │
  │                                        │                         │
  └─ release_workers() ──────────────────► fetch_add(available, N)  │
       db_private_free(this)               │                         │
```

---

## Lifecycle

| Stage | Code | Notes |
|-------|------|-------|
| Construction | `try_reserve_workers(N)` | `db_private_alloc` + `placement_new` |
| Active | `push_task()` × N | Increments `m_active_tasks` each call |
| Draining | `wait_workers()` | Spin until `m_active_tasks == 0` |
| Destruction | `release_workers()` → `db_private_free` | Explicit destructor + heap free |
| Cleanup via RAII | `~worker_manager()` calls `release_workers()` | Safety net if caller forgets |

---

## Constraints

- **Build mode**: `SERVER_MODE` or `SA_MODE` only (`#error` otherwise).
- **Allocation**: Object must live on the thread's private heap — never on the stack (atomics need stable address).
- **Ordering**: `push_task` must not be called with `m_active_tasks >= m_reserved_workers` (asserted).
- **`pop_task` ordering**: `pop_task()` must be called before `delete this` inside retire (the default retire lambda does this correctly).
- **No copy or assign**: deleted — one manager per parallel query phase, always heap-allocated.

---

## Related

- [[components/parallel-query|parallel-query]] — overview and degree selection
- [[components/parallel-worker-manager-global|parallel-worker-manager-global]] — the singleton being reserved against
- [[components/parallel-task-queue|parallel-task-queue]] — `callable_task` that calls `pop_task`
- [[components/entry-task|entry-task]] — `cubthread::entry_task` base class
- [[components/lockfree|lockfree]] — CAS primitives used in reservation loop
- [[Memory Management Conventions]] — `db_private_alloc` / `db_private_free` / `placement_new`
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
