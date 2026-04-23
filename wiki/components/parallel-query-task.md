---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_query_execute/px_query_task.{hpp,cpp}"
status: active
purpose: "Per-worker parallel subquery task: dequeues jobs, executes cloned XASL, reports results + errors"
key_files:
  - "px_query_task.hpp (task_args class, task_function free callable)"
  - "px_query_task.cpp (execute body, XASL clone handling)"
  - "px_query_job.hpp (job struct + join_context)"
public_api:
  - "parallel_query_execute::task_args class"
  - "parallel_query_execute::job struct (from px_query_job.hpp)"
  - "parallel_query_execute::join_context class (from px_query_job.hpp)"
tags:
  - component
  - cubrid
  - parallel
  - query
  - subquery
  - worker
related:
  - "[[components/parallel-query-executor|parallel-query-executor]]"
  - "[[components/parallel-query-checker|parallel-query-checker]]"
  - "[[components/parallel-query-execute|parallel-query-execute]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/parallel-interrupt|parallel-interrupt]]"
  - "[[components/xasl|xasl]]"
created: 2026-04-23
updated: 2026-04-23
---

# `parallel_query_execute::task_args` / `job` — Per-Worker Subquery Task

The worker-side implementation paired with [[components/parallel-query-executor|query_executor]]. Each spawned worker runs one `task_args`; each `task_args` dequeues `job` items from the shared queue until exhausted or interrupted.

## Files

| File | Role |
|------|------|
| `px_query_task.hpp` | `task_args` (worker context) |
| `px_query_task.cpp` | `task_args::execute` loop |
| `px_query_job.hpp` | `job` struct (POD — XASL_NODE* + xasl_state*), `join_context` |

## `job` — the unit of work

```cpp
namespace parallel_query_execute {
  struct job { xasl_node *xasl;  xasl_state *state; };  // conceptual
  class join_context { /* mutex + condvar + result set join point */ };
}
```

A `job` is lightweight — just pointers the worker needs to start executing an XASL. The worker does NOT own the XASL; it holds a reference the parent coordinator guarantees will outlive the queue drain.

## `task_args` — the worker-side struct

Bundles references the worker needs, taken from the parent `query_executor`:

```cpp
class task_args {
  using queue                 = parallel_query::thread_safe_queue<job>;
  using interrupt             = parallel_query::interrupt;
  using worker_manager        = parallel_query::worker_manager;
  using err_messages_with_lock = parallel_query::err_messages_with_lock;
public:
  // constructor takes queue*, interrupt*, worker_manager*, err_messages*, xasl_state*, ...
  // execute(): main loop
};
```

## Execution path — per worker

```
task_args constructed by query_executor
    │
    ├─► worker pool calls task_args::execute()
    │     │
    │     │ loop:
    │     │   ├─► dequeue job from queue
    │     │   │     │
    │     │   │     ├─► queue empty → exit loop
    │     │   │     └─► interrupt.is_set() → exit loop
    │     │   │
    │     │   ├─► clone xasl state for this worker (per-worker XASL visit state)
    │     │   │
    │     │   ├─► qexec_execute_xasl_proc (the serial executor, now in worker thread)
    │     │   │
    │     │   ├─► on error:
    │     │   │     ├─► er_stack_push_if_exists
    │     │   │     ├─► err_messages_with_lock.move_top_error_message_to_this()
    │     │   │     ├─► interrupt.set(ERROR_*)
    │     │   │     └─► exit loop
    │     │   │
    │     │   └─► on success: append to join_context output, continue
    │
    ├─► retire (entry_task contract)
    │     │
    │     │ release per-worker thread-local state
    │
    └─► worker pool reclaims task_args memory
```

## `join_context`

Shared synchronization anchor. Workers atomically append their output to the context; the main thread waits in [[components/parallel-query-executor|query_executor::run_jobs]] until the worker count reaches zero (yield-spin + optional condvar).

Internally holds:
- `std::mutex` for output list protection
- `std::condition_variable` for signal (optional — main thread may yield-spin instead)
- per-worker scoreboard

## Error propagation (critical detail)

CUBRID stores errors in `thread-local cuberr::context`. Workers CANNOT let that TL state die on retire — the main thread would never see the error. Each worker **must** call:

```cpp
err_messages_with_lock.move_top_error_message_to_this();
```

This swaps (not copies) the worker's top error into the shared list. See [[components/parallel-interrupt|parallel-interrupt]] for the `err_messages_with_lock` internals and why this is a move rather than a copy.

## Constraints

- **XASL cloning**: each worker needs its own `xasl_state` for private evaluation scratch (REGU_VARIABLE values, list scan IDs). The job provides pointers; task_args::execute is responsible for per-worker clone/reset.
- **Threading**: task_args executes in a worker thread; no access to the main thread's thread-local state.
- **Interrupt**: workers must poll `interrupt.is_set()` between jobs to respect cancellation.
- **Build-mode**: SERVER_MODE + SA_MODE only.
- **Lifetime**: `task_args` is `new`'d by query_executor and destroyed via the [[components/parallel-task-queue|callable_task]] retire convention.

## Lifecycle

1. `query_executor::run_jobs` pushes N `task_args` to the worker pool.
2. Each worker dequeues jobs until the queue drains or `interrupt` fires.
3. Each worker moves its last error (if any) into the shared `err_messages_with_lock`.
4. `retire()` runs — frees per-worker state and signals the task_queue.
5. Main thread in `query_executor::run_jobs` unblocks when all workers retire.
6. Main thread checks `err_messages` — rethrows first error or consumes results.

## Related

- Coordinator: [[components/parallel-query-executor|parallel-query-executor]]
- Hub: [[components/parallel-query-execute|parallel-query-execute]]
- Infrastructure: [[components/parallel-task-queue|task-queue]] (`callable_task`), [[components/parallel-interrupt|interrupt]], [[components/parallel-worker-manager|worker-manager]]
- Serial fallback: [[components/query-executor|query-executor]]
- XASL state: [[components/xasl|xasl]], [[components/regu-variable|regu-variable]]
