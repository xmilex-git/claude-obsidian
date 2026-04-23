---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_interrupt.hpp"
status: active
purpose: "Cross-thread interrupt signaling, ROWNUM early-exit counter, and mutex-protected error message collection for parallel query workers"
key_files:
  - "px_interrupt.hpp (interrupt, atomic_instnum, err_messages_with_lock — header-only)"
  - "error_context.hpp (cuberr::context, cuberr::er_message — CUBRID error layer)"
public_api:
  - "interrupt::get_code() -> interrupt_code"
  - "interrupt::set_code(interrupt_code)"
  - "interrupt::clear()"
  - "atomic_instnum::is_instnum_satisfies_after_1tuple_insert() -> bool"
  - "atomic_instnum::set_destination_tuple_cnt(size_t)"
  - "err_messages_with_lock::move_top_error_message_to_this() -> int"
tags:
  - component
  - cubrid
  - parallel
  - query
  - interrupt
  - error
related:
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/error-manager|error-manager]]"
  - "[[components/thread|thread]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_interrupt.hpp` — Parallel Query Interrupt and Error Propagation

`px_interrupt.hpp` defines three cooperating classes for coordinating early termination and error reporting across parallel worker threads. It is a **header-only** file; all methods are `inline`.

---

## Class / Function Inventory

### `interrupt` class

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `interrupt_code` | nested enum class | `enum class interrupt_code` | Typed interrupt code — 7 values |
| `m_code` | member | `std::atomic<interrupt_code>` | Atomic storage for the current interrupt state |
| `interrupt()` | constructor | `interrupt()` | Default: `m_code = NO_INTERRUPT` |
| `interrupt(code)` | constructor | `interrupt(interrupt_code)` | Initialised to a specific code |
| `get_code()` | inline const | `interrupt_code get_code() const noexcept` | `m_code.load(memory_order_acquire)` |
| `set_code(code)` | inline | `void set_code(interrupt_code) noexcept` | `m_code.store(code, memory_order_release)` |
| `clear()` | inline | `void clear() noexcept` | `m_code.store(NO_INTERRUPT, memory_order_release)` |

### `atomic_instnum` class

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `m_destination_tuple_cnt` | member | `std::size_t` | The ROWNUM limit — how many tuples before early exit |
| `m_current_tuple_cnt` | member | `std::atomic<std::size_t>` | Running count of tuples emitted across all workers |
| `m_is_instnum_set` | member | `bool` | Guard: true only if a ROWNUM limit was specified |
| `atomic_instnum()` | constructor | `atomic_instnum()` | All zero, `m_is_instnum_set = false` |
| `atomic_instnum(n)` | constructor | `atomic_instnum(std::size_t n)` | Sets destination count, `m_is_instnum_set = true` |
| `set_destination_tuple_cnt(n)` | inline | `void set_destination_tuple_cnt(std::size_t) noexcept` | Late-sets the limit (for deferred ROWNUM setup) |
| `is_instnum_satisfies_after_1tuple_insert()` | inline | `bool() noexcept` | `fetch_add(1) >= destination` — true means limit reached |

### `err_messages_with_lock` class

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `m_mutex` | member | `std::mutex` | Guards all access to `m_error_messages` |
| `m_error_messages` | member | `std::vector<cuberr::er_message*>` | Heap-allocated error message snapshots from workers |
| `err_messages_with_lock()` | constructor | default | Empty vector, default mutex |
| `~err_messages_with_lock()` | destructor | `~err_messages_with_lock()` | Deletes all `cuberr::er_message*` in the vector |
| `move_top_error_message_to_this()` | inline | `int move_top_error_message_to_this()` | Snapshots the calling thread's current error into the shared vector; returns `err_id` |

---

## `interrupt_code` — Complete Value List

```cpp
enum class interrupt_code {
    NO_INTERRUPT,                          // 0 — normal operation
    USER_INTERRUPTED_FROM_MAIN_THREAD,     // 1 — user cancelled (e.g. Ctrl-C) from main
    USER_INTERRUPTED_FROM_WORKER_THREAD,   // 2 — user cancel detected by a worker
    ERROR_INTERRUPTED_FROM_MAIN_THREAD,    // 3 — error detected by the main thread
    ERROR_INTERRUPTED_FROM_WORKER_THREAD,  // 4 — error detected by a worker thread
    INST_NUM_SATISFIED,                    // 5 — ROWNUM/INST_NUM limit reached
    JOB_ENDED,                             // 6 — all work dispatched (soft end signal)
};
```

The distinction between `_FROM_MAIN_THREAD` and `_FROM_WORKER_THREAD` variants allows the interrupt handler to identify the source of the interruption and take appropriate action (e.g. main-thread interrupts must be propagated to workers; worker-thread errors must be collected and replayed on the main thread).

---

## `interrupt` — Atomic Interrupt Code

```
interrupt
  └── m_code : std::atomic<interrupt_code>
        ├── get_code()   → load(acquire)
        ├── set_code()   → store(release)
        └── clear()      → store(NO_INTERRUPT, release)
```

The acquire/release pairing ensures: any memory writes performed before `set_code()` are visible to a thread that subsequently calls `get_code()` and observes a non-`NO_INTERRUPT` value.

The `interrupt` object is typically owned by the parallel query context (e.g. `parallel_heap_scan::manager`, `thread_safe_queue`). Workers and the main thread share a reference to the same instance.

### Usage in `thread_safe_queue`

Both `push_slow` and `pop_slow` accept a `const interrupt &interrupt_check`:

```cpp
while (is_full()) {
    if (interrupt_check.get_code() != interrupt::interrupt_code::NO_INTERRUPT)
        return;   // abort push immediately
    m_not_full.wait(lock);
}
```

This allows a blocked worker to be unblocked without the condvar being signaled — on the next wakeup (spurious or real), it checks the interrupt and returns.

---

## `atomic_instnum` — ROWNUM Early-Exit Mechanics

CUBRID supports `ROWNUM` predicates that limit the number of output rows. In parallel execution, multiple workers emit rows concurrently to a shared result buffer. `atomic_instnum` provides the shared counter:

```cpp
// Worker emits one tuple:
if (instnum.is_instnum_satisfies_after_1tuple_insert()) {
    // We just emitted the Nth tuple — trigger interrupt
    interrupt.set_code(interrupt_code::INST_NUM_SATISFIED);
    return;
}
```

The critical operation is `fetch_add(1)` compared against `m_destination_tuple_cnt`:

```cpp
inline bool is_instnum_satisfies_after_1tuple_insert() noexcept {
    return m_is_instnum_set
        ? m_current_tuple_cnt.fetch_add(1) >= m_destination_tuple_cnt
        : false;
}
```

`fetch_add` returns the value **before** the add. So the condition `>= destination` is true when the counter was already at or past the limit before this increment. The semantics are: "if, after this tuple, we have reached the limit, return true." Because `fetch_add` is atomic, exactly one worker will see the transition from `destination - 1` to `destination`; subsequent workers see `>= destination` immediately.

> [!key-insight] `fetch_add` not `compare_exchange` for ROWNUM
> `fetch_add(1)` is used rather than a CAS loop. This means it is possible for workers to emit slightly more than `destination` tuples in a race — multiple workers can simultaneously fetch values `>= destination`. This is an intentional trade-off: correctness of tuple count takes a back seat to performance. The query executor truncates the final result to the exact `ROWNUM` limit. The early-exit signal reduces unnecessary work without providing a hard guarantee on the exact number of tuples produced by workers.

> [!warning] `m_is_instnum_set` is not atomic
> The `m_is_instnum_set` flag is a plain `bool`. It must be set before any worker thread calls `is_instnum_satisfies_after_1tuple_insert()`. The construction pattern (setting `m_is_instnum_set` in the constructor or via `set_destination_tuple_cnt`) must happen-before the workers start — typically satisfied by the reservation protocol (all setup before `push_task`).

---

## `err_messages_with_lock` — Cross-Thread Error Propagation

### The Problem: Thread-Local Error Context

CUBRID's error system uses a thread-local error context (`cuberr::context::get_thread_local_context()`). Each thread has its own error stack. When a worker thread encounters an error and sets it via `er_set(...)`, that error is only visible on the worker thread's local context. The main thread's context has no knowledge of it.

```
Worker thread         Main thread
  │                       │
  er_set(ER_DISK_FULL)    │
  thread_local error ─╳──►│   (not visible — different thread-local)
  │                       │
  err_messages_with_lock::move_top_error_message_to_this()
  │ snapshot the error ──►│  (now main thread can replay it)
```

### `move_top_error_message_to_this`

```cpp
inline int move_top_error_message_to_this() {
    int err_id = 0;
    std::lock_guard<std::mutex> lock(m_mutex);

    // Allocate a new er_message on the heap (false = not in arena)
    m_error_messages.push_back(new cuberr::er_message(false));

    // Get the error ID from the current thread's error context
    err_id = cuberr::context::get_thread_local_context()
                             .get_current_error_level().err_id;

    // Swap (move) the current thread's error into the heap-allocated message
    m_error_messages.back()->swap(
        cuberr::context::get_thread_local_context()
                        .get_current_error_level());

    return err_id;
}
```

The `swap` moves the error message content from the thread-local context into the heap-allocated `er_message`, leaving the thread-local context clear. After this call, the worker's thread-local error is gone, and the error lives in `m_error_messages` where the main thread can access it under the mutex.

> [!key-insight] Why `swap` instead of copy?
> `er_message::swap` moves ownership of the message string and any associated fields rather than copying them. This avoids a string allocation and prevents the thread-local context from holding a stale copy of the error after the worker exits. The worker's error is consumed, not duplicated.

### Lifetime

- `err_messages_with_lock` is typically owned by the parallel query context.
- Worker threads call `move_top_error_message_to_this()` before retiring if an error occurred.
- The main thread iterates `m_error_messages` after all workers complete to find and re-raise errors.
- The destructor `delete`s all stored `er_message*` pointers.

> [!warning] Error message ownership transfer is one-way
> Once `move_top_error_message_to_this()` swaps the error out of the thread-local context, the worker thread can no longer access it. The worker should not use `er_errid()` or `er_msg()` after the call — those APIs read from the (now-cleared) thread-local context.

---

## Execution Path — Interrupt and Error Flow

```
Main thread                          Worker thread
  │                                       │
  │   sets up interrupt object            │
  │   sets up atomic_instnum              │
  │   sets up err_messages_with_lock      │
  │                                       │
  │   push_task(callable_task)            │
  │                                       ├─ execute()
  │                                       │    ├─ process tuples
  │                                       │    ├─ is_instnum_satisfies_after_1tuple_insert()
  │                                       │    │     └─ if true: set_code(INST_NUM_SATISFIED)
  │                                       │    │         └─ return early
  │                                       │    ├─ interrupt.get_code() != NO_INTERRUPT
  │                                       │    │     └─ return early
  │                                       │    └─ on error: move_top_error_message_to_this()
  │                                       ├─ retire()
  │                                       │    └─ pop_task()
  │                                       │
  ├─ wait_workers() [spin]                │
  ◄─ active_tasks == 0                    │
  │                                       │
  ├─ check interrupt code                 │
  ├─ replay errors from m_error_messages  │
  └─ continue                             │
```

---

## Constraints

- `px_interrupt.hpp` has no build-mode guard — it is header-only and does not include any server-specific headers directly (only `<atomic>`, `<mutex>`, `<vector>`, `error_context.hpp`).
- `atomic_instnum::m_is_instnum_set` is a plain bool — must be written before worker threads start (not thread-safe itself).
- `err_messages_with_lock::move_top_error_message_to_this()` must be called from a worker thread that has an active error in its thread-local context — calling it with no error produces a no-op message (err_id = 0).
- The mutex in `err_messages_with_lock` is not recursive; do not call from within a held lock.

---

## Related

- [[components/parallel-query|parallel-query]] — interrupt context in the broader subsystem
- [[components/parallel-task-queue|parallel-task-queue]] — `thread_safe_queue` passes `interrupt&` to slow-path push/pop
- [[components/error-manager|error-manager]] — `cuberr::context`, `er_set`, thread-local error stack
- [[components/thread|thread]] — `cubthread::entry` where thread-local context lives
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
