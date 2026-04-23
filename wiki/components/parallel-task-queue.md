---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/"
status: developing
purpose: "Thread-safe MPMC slot queue and std::function-based callable_task abstraction for parallel query workers"
key_files:
  - "px_thread_safe_queue.hpp (thread_safe_queue<T> template — MPMC with slot sequences)"
  - "px_callable_task.hpp (callable_task — std::function wrapper over cubthread::task<entry>)"
  - "px_interrupt.hpp (interrupt used by queue push/pop as stop signal)"
public_api:
  - "thread_safe_queue<T>::push(value, interrupt)"
  - "thread_safe_queue<T>::pop(value, interrupt) -> bool"
  - "thread_safe_queue<T>::try_push(value) -> bool"
  - "thread_safe_queue<T>::try_pop(value) -> bool"
  - "thread_safe_queue<T>::push_last() (signals end-of-stream)"
  - "callable_task(worker_manager*, F, delete_on_retire)"
  - "callable_task(worker_manager*, FuncExec, FuncRetire)"
tags:
  - component
  - cubrid
  - parallel
  - query
related:
  - "[[modules/src|src]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_thread_safe_queue` + `px_callable_task` — Task Queue Primitives

Two complementary primitives that bridge the gap between parallel query logic and CUBRID's `cubthread` infrastructure.

## `thread_safe_queue<T>`

A templated **multi-producer multi-consumer (MPMC)** queue backed by a slot array. Defined in `px_thread_safe_queue.hpp`.

### Internal Structure

```
thread_safe_queue<T>
  ├── m_slots         : std::vector<slot<T>>         (ring buffer)
  │     slot<T>
  │       ├── data     : T
  │       ├── sequence : std::atomic<uint64_t>       (MPMC sequencing)
  │       └── ready    : std::atomic<bool>
  ├── m_enqueue_pos   : std::atomic<uint64_t>
  ├── m_dequeue_pos   : std::atomic<uint64_t>
  ├── m_push_completed: std::atomic<bool>            (end-of-stream flag)
  ├── m_resetting     : std::atomic<bool>
  ├── m_capacity      : size_t
  ├── m_mutex         : std::mutex                   (slow-path only)
  ├── m_not_empty     : std::condition_variable
  └── m_not_full      : std::condition_variable
```

### Fast vs Slow Path

| Method | Fast path | Slow path |
|--------|-----------|-----------|
| `push` | `try_push_fast` (CAS on sequence) | `push_slow` (mutex + `m_not_full` wait, interrupt-aware) |
| `pop` | `try_pop_fast` (CAS on sequence) | `pop_slow` (mutex + `m_not_empty` wait, interrupt-aware) |

> [!key-insight] Interrupt-aware blocking
> Both `push` and `pop` accept a `const interrupt &interrupt_check`. The slow path checks the interrupt code while waiting on the condition variable, allowing a blocked worker thread to be unblocked when the main thread sets an interrupt code (e.g. `USER_INTERRUPTED_FROM_MAIN_THREAD`).

### End-of-Stream

`push_last()` sets `m_push_completed = true` and notifies all waiters. Consumers see this as the signal that no more items will arrive.

### Reset

`reset_queue()` resets positions and slot states for queue reuse. The `m_resetting` flag prevents concurrent push/pop during reset.

## `callable_task`

Defined in `px_callable_task.hpp`. A concrete `cubthread::task<cubthread::entry>` that wraps an arbitrary `std::function<void(cubthread::entry&)>` as the execution body.

### Class Layout

```
callable_task : cubthread::task<cubthread::entry>
  ├── m_exec_f    : std::function<void(cubthread::entry&)>
  ├── m_retire_f  : std::function<void(void)>
  └── m_worker_manager_p : worker_manager*
```

### Constructors

| Signature | retire behaviour |
|-----------|-----------------|
| `callable_task(wm, f, delete_on_retire=true)` | default: `[this]{ delete this; }` |
| `callable_task(wm, f, delete_on_retire=false)` | no-op retire (caller manages lifetime) |
| `callable_task(wm, fe, fr)` | explicit custom retire function |

Both copy and move variants of each constructor are provided.

### execute / retire

```cpp
void execute(cubthread::entry &context) {
    m_exec_f(context);
}
void retire() {
    m_worker_manager_p->pop_task();   // decrement active count
    m_retire_f();                     // default: delete this
}
```

> [!key-insight] pop_task is called in retire, not execute
> The active task count (`worker_manager::m_active_tasks`) is decremented inside `retire()`, not `execute()`. CUBRID's worker pool calls `retire()` after `execute()` completes, so the task object must stay valid until `retire()` runs — which is why the default retire deletes `this` after `pop_task()`.

## Usage in Parallel Sort

`px_sort.h` uses `callable_task` directly via macro:
```cpp
#define SORT_EXECUTE_PARALLEL(num, px_sort_param, function)  \
    for (int i = 0; i < num; i++) {                          \
        auto *task = new callable_task(wm, std::bind(function, _1, &px_sort_param[i])); \
        wm->push_task(task);                                  \
    }
```

## Related

- [[components/parallel-query|parallel-query]] — parent overview
- [[components/parallel-worker-manager|parallel-worker-manager]] — manages the pool that runs these tasks
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
