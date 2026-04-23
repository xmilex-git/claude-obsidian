---
type: component
parent_module: "[[modules/src|src]]"
path: "src/thread/"
status: stable
purpose: "Abstract base class for worker-pool tasks carrying a THREAD_ENTRY context; defines the execute/retire lifecycle used throughout the server"
key_files:
  - "thread_task.hpp (task<Context>, callable_task<Context>, task<void>)"
  - "thread_entry_task.hpp (entry_task alias, entry_manager, daemon_entry_manager, system_worker_entry_manager)"
public_api:
  - "cubthread::entry_task = task<entry> — base class for all pool tasks"
  - "entry_task::execute(entry&) — pure virtual; implement work here"
  - "entry_task::retire() — default: delete this; override for pooling"
  - "cubthread::entry_callable_task = callable_task<entry> — std::function wrapper"
  - "cubthread::entry_manager — base context lifecycle manager"
  - "entry_manager::create_context() / retire_context() / recycle_context()"
  - "cubthread::daemon_entry_manager — entry_manager specialization for daemons"
  - "cubthread::system_worker_entry_manager — sets thread_type + txn index on create/recycle"
  - "cubthread::task<void> (= task_without_context) — context-less daemon tasks"
tags:
  - component
  - cubrid
  - thread
  - concurrency
related:
  - "[[components/thread|thread]]"
  - "[[components/worker-pool|worker-pool]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[components/thread-daemon|thread-daemon]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/vacuum|vacuum]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cubthread::entry_task` — Entry Task Abstract Base

`entry_task` is the fundamental unit of work in the `cubthread` worker pool system. It is a `task<entry>` — a template specialization of `cubthread::task<Context>` where the context is a `THREAD_ENTRY`.

## Generic Task Template

`thread_task.hpp` defines the generic base:

```cpp
template <typename Context>
class task {
public:
  virtual void execute(Context&) = 0;  // implement the work
  virtual void retire() { delete this; }  // default: self-delete after execution
};

// Context-less specialization for daemon tasks
template<>
class task<void> {
public:
  virtual void execute() = 0;
  virtual void retire() { delete this; }
};
using task_without_context = task<void>;
```

`entry_task` is simply:
```cpp
using entry_task = task<entry>;
```

## The Retire Pattern

`retire()` is called by the worker pool after `execute()` completes. The **default** implementation is `delete this` — the task self-destructs.

Subclasses override `retire()` to implement alternative lifetime strategies:

| Pattern | When to use |
|---------|-------------|
| `delete this` (default) | Short-lived, heap-allocated task — common case |
| No-op `retire()` | Task is stack-allocated or managed externally |
| Custom retire | Task notifies parent (e.g., `worker_manager::pop_task()`) and then deletes |
| Return to pool | Long-lived, re-submitted tasks with pre-allocated storage |

> [!key-insight] retire() is how parallel-query counts completion
> In [[components/parallel-worker-manager|parallel-worker-manager]], each task's `retire()` calls `worker_manager::pop_task()` which does `m_active_tasks.fetch_sub(1)`. The main thread spins in `wait_workers()` until `m_active_tasks == 0`. The retire-and-decrement pattern avoids any explicit completion event — the task cleans itself up and signals the barrier atomically.

## `callable_task<Context>` — Lambda Wrapper

For ad-hoc tasks without defining a subclass:

```cpp
using entry_callable_task = callable_task<entry>;

// Usage
auto* task = new entry_callable_task(
    [](cubthread::entry& ctx) { /* work */ },          // execute
    [task]() { task->pop_from_manager(); delete task; } // retire (optional)
);
```

The two-argument constructor allows independent execute and retire lambdas. The single-argument constructor uses `delete this` or a no-op for retire based on `delete_on_retire` bool.

## `entry_manager` — Context Lifecycle

`entry_manager` pools and prepares the `THREAD_ENTRY` context for worker threads. It wraps `cubthread::manager`'s `claim_entry`/`retire_entry`:

```cpp
class entry_manager {
public:
  virtual entry& create_context();      // claim entry from pool + on_create()
  virtual void   retire_context(entry&); // on_retire() + return entry to pool
  virtual void   recycle_context(entry&);// on_recycle() — called between tasks on same worker
  virtual void   stop_execution(entry&); // interrupt running task
protected:
  virtual void on_create(entry&) {}     // override: set up entry for this pool type
  virtual void on_retire(entry&) {}     // override: tear down (mirror of on_create)
  virtual void on_recycle(entry&) {}    // override: reset between tasks
};
```

### Specialized Entry Managers

| Class | Purpose |
|-------|---------|
| `daemon_entry_manager` | Adds `on_daemon_create`/`on_daemon_retire` hooks; `on_recycle` is no-op (daemons don't recycle) |
| `system_worker_entry_manager` | Sets `entry.type` to a specified `thread_type` and assigns a dummy `tran_index` on create/recycle — used by system-internal pools that need a non-null transaction index for perf logging |

## How to Add a New Pool Task

1. Subclass `entry_task`:
   ```cpp
   class my_task : public cubthread::entry_task {
   public:
     void execute(cubthread::entry& thread_ref) override { /* work */ }
     // optionally override retire() for custom cleanup
   };
   ```
2. Subclass `entry_manager` if the pool needs custom context setup.
3. Register the pool with `REGISTER_WORKERPOOL(name, size_getter)`.
4. Create the pool via `cubthread::get_manager()->create_worker_pool<worker_pool_type>(...)`.
5. Dispatch tasks via `cubthread::get_manager()->push_task(pool, new my_task())`.

## Daemon Tasks vs Pool Tasks

| Aspect | Pool task (entry_task) | Daemon task (entry_task or task<void>) |
|--------|----------------------|----------------------------------------|
| Context | `THREAD_ENTRY&` | `THREAD_ENTRY&` (or none for `task<void>`) |
| Lifecycle | One-shot: execute once, retire | Looped: execute repeatedly per looper tick |
| Allocation | Heap per dispatch | Usually one persistent instance per daemon |
| Retire | Called once after execute | NOT called between loops; called on daemon stop |

## Related

- [[components/thread|thread]] — parent namespace overview
- [[components/worker-pool|worker-pool]] — pool that executes entry_task instances
- [[components/thread-daemon|thread-daemon]] — daemons that use entry_task for loop body
- [[components/parallel-task-queue|parallel-task-queue]] — parallel-query uses `callable_task<entry>` wrapper
- [[components/vacuum|vacuum]] — vacuum tasks are entry_task subclasses
- Source: [[sources/cubrid-src-thread|cubrid-src-thread]]
