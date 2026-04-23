---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/"
status: active
purpose: "Thread-safe MPMC slot queue (thread_safe_queue<T>) and std::function-based callable_task abstraction for parallel query workers"
key_files:
  - "px_thread_safe_queue.hpp / px_thread_safe_queue.cpp (MPMC slot queue with fast+slow paths)"
  - "px_callable_task.hpp / px_callable_task.cpp (callable_task — entry_task wrapping std::function)"
  - "px_interrupt.hpp (interrupt used by queue push/pop as stop signal)"
public_api:
  - "thread_safe_queue<T>::push(value, interrupt)"
  - "thread_safe_queue<T>::pop(value, interrupt) -> bool"
  - "thread_safe_queue<T>::try_push(value) -> bool"
  - "thread_safe_queue<T>::try_pop(value) -> bool"
  - "thread_safe_queue<T>::push_last()"
  - "thread_safe_queue<T>::reset_queue()"
  - "callable_task(worker_manager*, F, delete_on_retire=true)"
  - "callable_task(worker_manager*, FuncExec, FuncRetire)"
tags:
  - component
  - cubrid
  - parallel
  - query
related:
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/parallel-interrupt|parallel-interrupt]]"
  - "[[components/entry-task|entry-task]]"
  - "[[components/thread|thread]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_thread_safe_queue` + `px_callable_task` — Task Queue Primitives

Two complementary primitives that bridge the gap between parallel query logic and CUBRID's `cubthread` infrastructure.

---

## Part 1: `thread_safe_queue<T>`

Defined in `px_thread_safe_queue.hpp`. A templated **multi-producer multi-consumer (MPMC)** ring buffer with a lock-free fast path and a condvar-based slow path.

### Class / Function Inventory

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `thread_safe_queue` | constructor | `explicit thread_safe_queue(std::size_t capacity = 1024)` | Initialises slot array, positions = 0 |
| `~thread_safe_queue` | destructor | `~thread_safe_queue()` | Takes mutex, notifies all waiters (unblocks any blocked push/pop) |
| `push` | method | `void push(const T&, const interrupt& = interrupt())` | Fast path first; falls through to slow path |
| `pop` | method | `bool pop(T&, const interrupt& = interrupt())` | Fast path first; falls through to slow path; returns false on end-of-stream or interrupt |
| `try_push` | method | `bool try_push(const T&)` | Fast path only — returns `false` immediately if full or resetting |
| `try_pop` | method | `bool try_pop(T&)` | Fast path only — returns `false` immediately if empty or resetting |
| `push_last` | method | `void push_last()` | Sets `m_push_completed = true`, notifies all — end-of-stream signal |
| `is_empty` | const | `bool is_empty() const` | `dequeue_pos == enqueue_pos` (approximate under concurrency) |
| `is_full` | const | `bool is_full() const` | `(enqueue_pos - dequeue_pos) >= capacity` |
| `size` | const | `std::size_t size() const` | `enqueue_pos - dequeue_pos` |
| `capacity` | const | `std::size_t capacity() const` | Returns `m_capacity` |
| `reset_queue` | method | `void reset_queue()` | CAS `m_resetting` true, then lock, recompute positions, reset slot sequences |
| `try_push_fast` | private | `bool try_push_fast(const T&)` | Lock-free CAS on slot sequence; increments `m_enqueue_pos` |
| `try_pop_fast` | private | `bool try_pop_fast(T&)` | Lock-free CAS on slot sequence; increments `m_dequeue_pos` |
| `push_slow` | private | `void push_slow(const T&, const interrupt&)` | Mutex + `m_not_full` condvar wait; interrupt-aware |
| `pop_slow` | private | `bool pop_slow(T&, const interrupt&)` | Mutex + `m_not_empty` condvar wait; interrupt-aware |

### `slot<T>` struct

```cpp
template<typename T>
struct slot {
    T data;
    std::atomic<std::uint64_t> sequence;   // MPMC sequencing token
    std::atomic<bool> ready;               // true once data is written and fence executed

    slot() : sequence(0), ready(false) {}
};
```

### Internal Structure

```
thread_safe_queue<T>
  ├── m_slots          : std::vector<slot<T>>        ring buffer (size capped at DB_UINT16_MAX)
  ├── m_enqueue_pos    : std::atomic<uint64_t>        monotonically increasing enqueue cursor
  ├── m_dequeue_pos    : std::atomic<uint64_t>        monotonically increasing dequeue cursor
  ├── m_push_completed : std::atomic<bool>            end-of-stream flag (set by push_last)
  ├── m_resetting      : std::atomic<bool>            reset-in-progress guard
  ├── m_capacity       : size_t                       actual ring buffer size
  ├── m_mutex          : std::mutex                   slow-path guard
  ├── m_not_empty      : std::condition_variable      wakes blocked consumers
  └── m_not_full       : std::condition_variable      wakes blocked producers
```

> [!key-insight] Why a fixed-slot ring buffer and not an unbounded queue?
> The parallel query subsystem operates on bounded work — a known number of partitions or pages. An unbounded queue would require dynamic allocation per item, adding allocator pressure on hot paths. The slot array is allocated once at construction, and all push/pop operations only CAS slot sequence numbers. This keeps the hot path allocation-free.

---

### MPMC Slot Protocol — ABA Prevention

The sequence numbers are the key to correct MPMC operation without ABA problems:

**At construction:** `slot[i].sequence = i` (each slot starts at its own index)

**Push sequence for slot at position `pos` (slot index = `pos % capacity`):**

1. CAS `slot.sequence` from `pos` → `pos + capacity`
2. Write `slot.data`
3. `atomic_thread_fence(release)`
4. `slot.ready.store(true, release)`
5. `m_enqueue_pos.fetch_add(1, release)`

**Pop sequence for slot at position `pos`:**

1. Check `slot.ready.load(acquire)` — if false, slot not yet written, bail out
2. CAS `slot.sequence` from `pos + capacity` → `pos + 2*capacity`
3. `atomic_thread_fence(acquire)`
4. Read `slot.data`
5. `slot.ready.store(false, release)`
6. `m_dequeue_pos.fetch_add(1, release)`

Each slot cycles through sequence values: `i`, `i + capacity`, `i + 2*capacity`, ... The distance between push-expected (`pos`) and pop-expected (`pos + capacity`) is exactly `capacity`, so a stale slot from a previous cycle has sequence `pos - capacity` which never matches the expected values. This is the classic Dmitry Vyukov MPMC ring buffer ABA solution.

> [!key-insight] `ready` flag as secondary guard
> The `ready` atomic bool is a belt-and-suspenders addition beyond the sequence number. Even if a consumer wins the sequence CAS, it must still observe `ready == true` before reading data. This protects against the narrow window where the producer CASed the sequence but hasn't yet stored `slot.data` (which can happen on arm64 if the thread is preempted between the CAS and the write — though the release fence should prevent this in practice).

---

### Overflow and Reset

Both `try_push_fast` and `try_pop_fast` check for uint64 overflow:

```cpp
// push: if pos near UINT64_MAX
if (pos > UINT64_MAX - m_capacity) {
    reset_queue();
    pos = 0;
}
// pop: similar check with 2*capacity margin
```

`reset_queue()` uses a CAS on `m_resetting` to ensure only one reset runs at a time, then takes the mutex to prevent concurrent pushes/pops during the reset. It modulo-reduces both position counters and resets slot sequences, then notifies all waiters.

There is also a subtle "epoch bump" after position wrap: after `fetch_add(1)`, if `pos % capacity == 0 && pos != 0`, an additional `fetch_add(capacity)` is performed. This skips over the modulo-zero position to avoid colliding sequence expectations on the next cycle.

---

### Fast vs Slow Path

| Operation | Fast Path | Slow Path |
|-----------|-----------|-----------|
| `push` | CAS on sequence, no lock | `std::unique_lock` + `m_not_full.wait`, interrupt-aware |
| `pop` | CAS on sequence, no lock | `std::unique_lock` + `m_not_empty.wait`, interrupt-aware |

The slow path is entered only when the queue is full (push) or empty (pop). Both slow paths accept an `interrupt&` and check it in the wait loop — a worker can be unblocked by setting the interrupt code from another thread.

---

### End-of-Stream

```cpp
void push_last() {
    m_push_completed.store(true, std::memory_order_release);
    std::lock_guard<std::mutex> lock(m_mutex);
    m_not_empty.notify_all();          // wake all blocked consumers
}
```

After `push_last()`, all subsequent `pop` calls return `false` once the queue is empty.

---

### Template Instantiation

Only one explicit instantiation exists in the `.cpp` file:

```cpp
template class parallel_query::thread_safe_queue<parallel_query_execute::job>;
```

The queue is currently used only for the parallel query execute job queue. Hash join and heap scan do not use this queue (they use `worker_manager::push_task` directly).

---

## Part 2: `callable_task`

Defined in `px_callable_task.hpp`. A concrete `cubthread::task<cubthread::entry>` that wraps an arbitrary `std::function<void(cubthread::entry&)>` as the execution body.

### Class / Function Inventory

| Symbol | Kind | Signature | Role |
|--------|------|-----------|------|
| `callable_task(wm, f, del=true)` | constructor (copy) | `callable_task(worker_manager*, const F&, bool)` | Stores f as `exec_func`, sets retire to `delete this` or no-op |
| `callable_task(wm, f, del=true)` | constructor (move) | `callable_task(worker_manager*, F&&, bool)` | Move-constructs `m_exec_f` from rvalue |
| `callable_task(wm, fe, fr)` | constructor (copy) | `callable_task(worker_manager*, const FE&, const FR&)` | Custom retire function (copy) |
| `callable_task(wm, fe, fr)` | constructor (move) | `callable_task(worker_manager*, FE&&, FR&&)` | Custom retire function (move) |
| `execute` | override | `void execute(cubthread::entry&)` | Calls `m_exec_f(context)` — asserts `m_worker_manager_p != nullptr` |
| `retire` | override | `void retire()` | Calls `pop_task()`, nulls pointer, then calls `m_retire_f()` |

### Class Layout

```
callable_task : cubthread::task<cubthread::entry>
  ├── m_exec_f           : std::function<void(cubthread::entry&)>
  ├── m_retire_f         : std::function<void(void)>
  └── m_worker_manager_p : worker_manager*
```

### Default Retire Behaviour

```cpp
// delete_on_retire = true (default):
m_retire_f = [this] { delete this; };

// delete_on_retire = false:
m_retire_f = [] {};  // caller manages lifetime
```

### `execute` and `retire` Implementation

```cpp
void callable_task::execute(cubthread::entry &context) {
    assert(m_worker_manager_p != nullptr);
    m_exec_f(context);
}

void callable_task::retire() {
    assert(m_worker_manager_p != nullptr);
    m_worker_manager_p->pop_task();     // step 1: decrement active task counter
    m_worker_manager_p = nullptr;       // step 2: prevent double-decrement
    m_retire_f();                       // step 3: default = delete this
}
```

> [!key-insight] `pop_task` is called in `retire`, not `execute`
> The active task count (`worker_manager::m_active_tasks`) is decremented inside `retire()`, not `execute()`. CUBRID's worker pool calls `retire()` after `execute()` completes. Decrementing before `delete this` is essential: if the decrement happened inside `execute`, a racing `wait_workers` might observe 0 and proceed to `release_workers` before `retire` runs, causing a use-after-free when `retire` later tries to access the already-freed task object.

> [!warning] `m_worker_manager_p` nulled before `m_retire_f()`
> The pointer is nulled between `pop_task()` and the retire function call. This prevents a second call to `pop_task()` if `m_retire_f` somehow re-enters `retire` (defensive programming). It also means any code in a custom retire function cannot use the pointer.

### Build Mode Guard

`px_callable_task.hpp` enforces `SERVER_MODE` or `SA_MODE`:

```cpp
#if !defined(SERVER_MODE) && !defined(SA_MODE)
#error Wrong module
#endif
```

---

## Usage in Parallel Sort

`px_sort.h` uses `callable_task` directly via macro:

```cpp
#define SORT_EXECUTE_PARALLEL(num, px_sort_param, function)  \
    do {                          \
        for (int i = 0; i < num; i++) {                                     \
            parallel_query::callable_task *task =                           \
               new parallel_query::callable_task(sort_param->px_worker_manager, \
                  std::bind(function, std::placeholders::_1, &px_sort_param[i])); \
            sort_param->px_worker_manager->push_task(task);                 \
        }                                                                   \
    } while (0)
```

Each `callable_task` wraps `std::bind(function, _1, &px_sort_param[i])`, fixing the per-partition parameter while leaving the `cubthread::entry&` argument to be supplied by the pool worker. Default `delete_on_retire = true` is used, so tasks self-destruct after completion.

---

## Lifecycle

### `callable_task` task dispatch → execute → retire

```
caller
  └─ new callable_task(wm, fn)          heap allocation (new)
       │
       └─ wm->push_task(task)
            ├─ m_active_tasks.fetch_add(1)
            └─ global_pool->push_task(task)
                    │
                    └─ [pool worker thread]
                         ├─ task->execute(entry)   m_exec_f(context)
                         └─ task->retire()
                              ├─ pop_task()         m_active_tasks.fetch_sub(1)
                              ├─ m_worker_manager_p = nullptr
                              └─ m_retire_f()       delete this  (default)
```

---

## Constraints

- `callable_task` default constructor is deleted — always requires a `worker_manager*`.
- `m_worker_manager_p` must not be `nullptr` at time of `execute` or `retire` (asserted).
- `callable_task` is heap-allocated when `delete_on_retire = true` (the default). Never stack-allocate in that case.
- `thread_safe_queue` capacity is capped at `DB_UINT16_MAX` (65535) regardless of the argument passed to the constructor.

---

## Related

- [[components/parallel-query|parallel-query]] — degree selection and subsystem overview
- [[components/parallel-worker-manager|parallel-worker-manager]] — the manager that `callable_task` reports back to
- [[components/parallel-interrupt|parallel-interrupt]] — `interrupt` class used by queue's slow paths
- [[components/entry-task|entry-task]] — `cubthread::entry_task` base class hierarchy
- [[components/thread|thread]] — `cubthread::entry` context passed to `execute`
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
