---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_query_execute/px_query_executor.{hpp,cpp}"
status: active
purpose: "Parallel subquery coordinator: owns the job queue + per-query worker manager, dispatches XASL-clone tasks, aggregates results"
key_files:
  - "px_query_executor.hpp (query_executor class, parent-chain ctor, add_job/run_jobs API)"
  - "px_query_executor.cpp (job queue management, worker spawning, result aggregation)"
public_api:
  - "parallel_query_execute::query_executor(root_thread_p, worker_manager*, parallelism, estimated_jobs, on_trace, xasl_state*)"
  - "query_executor(query_executor *parent) — nested parallel subquery ctor"
  - "bool add_job(thread_p, xasl, xasl_state)"
  - "int run_jobs(thread_p)"
  - "int get_parallelism() const"
  - "query_executor_stats get_stats() const"
tags:
  - component
  - cubrid
  - parallel
  - query
  - subquery
  - coordinator
related:
  - "[[components/parallel-query-execute|parallel-query-execute]]"
  - "[[components/parallel-query-checker|parallel-query-checker]]"
  - "[[components/parallel-query-task|parallel-query-task]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/parallel-interrupt|parallel-interrupt]]"
  - "[[components/xasl|xasl]]"
created: 2026-04-23
updated: 2026-04-23
---

# `query_executor` — Parallel Subquery Coordinator

Owns the `thread_safe_queue<job>`, the per-query [[components/parallel-worker-manager|worker_manager]], and the interrupt machinery for one parallel subquery. Lives in the `parallel_query_execute` namespace.

## Type aliases (from the class body)

```cpp
using queue = parallel_query::thread_safe_queue<job>;
using worker_manager = parallel_query::worker_manager;
using err_messages_with_lock = parallel_query::err_messages_with_lock;
using interrupt = parallel_query::interrupt;
using query_executor_stats = XASL_STATS;
```

Aliases let the file reuse generic parallel infrastructure from [[components/parallel-task-queue|task-queue]] / [[components/parallel-worker-manager|worker-manager]] / [[components/parallel-interrupt|interrupt]] with type-specialized names (`job` here).

## Public API

```cpp
query_executor (THREAD_ENTRY *root_thread_p,
                worker_manager *worker_manager_p,
                int parallelism,
                int estimated_jobs,
                bool on_trace,
                xasl_state *xasl_state);

query_executor (query_executor *parent_executor_p);  // nested-parallel ctor

~query_executor();

bool add_job (THREAD_ENTRY *thread_p, xasl_node *xasl, xasl_state *xasl_state);
int  run_jobs (THREAD_ENTRY *thread_p);
int  get_parallelism() const;
query_executor_stats get_stats() const;
```

### Ctor forms

- **Root ctor**: takes `root_thread_p` + a pre-reserved `worker_manager*` + degree + estimated job count + trace flag + `xasl_state`. Standard entry point from [[components/query-executor|qexec_execute_mainblock]].
- **Parent ctor**: `query_executor(query_executor *parent)` — nested parallel case. When a parallel subquery contains **another** parallel subquery, the inner executor borrows worker-manager machinery from the parent (shared pool, separate job queue). Prevents pool exhaustion.

## Execution path — `add_job` → `run_jobs`

```
query_executor ctor
    │
    │  on_scan_start: reserve N workers from global pool
    │
    ├─► add_job(thread_p, xasl_n, state) × K
    │     │
    │     │  enqueue(job{xasl_n, state}) into thread_safe_queue
    │     │  return true if queued, false if queue full or interrupted
    │
    ├─► run_jobs(thread_p)
    │     │
    │     │  push N × parallel_query_execute::task_args → [[components/parallel-query-task|task]]
    │     │  workers dequeue jobs until queue empty
    │     │  each task executes xasl in a cloned thread_entry
    │     │  trace data (if on_trace) accumulated into query_executor_stats
    │     │
    │     ├─► wait_workers() (yield-spin until all done)
    │     │
    │     └─► check err_messages_with_lock — rethrow first error
    │
    └─► dtor
          │
          │  release_workers back to global pool
```

## Key fields (private — inferred)

- `queue m_job_queue` — producer/consumer queue, pre-sized from `estimated_jobs`
- `worker_manager *m_worker_manager` — per-query reservation (borrowed or owned)
- `interrupt m_interrupt` — cross-worker cancel + err propagation
- `err_messages_with_lock m_errs` — worker errors aggregated here
- `query_executor_stats m_stats` — XASL_STATS snapshot for trace/EXPLAIN
- `int m_parallelism` — stored degree
- `bool m_on_trace` — whether trace is enabled
- `xasl_state *m_xasl_state` — shared XASL runtime state

## Constraints

- **Threading**: thread-safe queue for dispatch; main thread owns stats + interrupt; workers produce rows + error snapshots.
- **Build-mode**: SERVER_MODE + SA_MODE only.
- **Memory**: `query_executor` itself is stack or `db_private_alloc`'d; all jobs live in the queue; tasks are `new`'d and retired by [[components/parallel-task-queue|callable_task]] convention.
- **Nested parallelism**: parent ctor explicitly permits nesting; recursion depth limited by `PRM_ID_PARALLELISM` and system-wide `PRM_ID_MAX_PARALLEL_WORKERS`.

## Lifecycle

1. `query_executor` constructed by [[components/query-executor|query-executor]] after `check_parallel_subquery_possible` returned a degree > 0.
2. Caller adds 1..K jobs via `add_job`.
3. Caller calls `run_jobs` — blocks (yield-spin) until all workers finish.
4. Caller reads stats + result list files written by workers.
5. Destructor releases the worker pool reservation.

## Integration with trace / EXPLAIN

When `on_trace = true`, workers write to an internal trace structure aggregated into `query_executor_stats` at teardown. Rendered by [[components/query-dump|query-dump]] for EXPLAIN ANALYZE output. Contrast with heap-scan trace which uses a dedicated [[components/parallel-heap-scan-support|trace_storage_for_sibling_xasl]] Jansson aggregator.

## Related

- Feasibility gate: [[components/parallel-query-checker|parallel-query-checker]]
- Worker task: [[components/parallel-query-task|parallel-query-task]]
- Infrastructure reuse: [[components/parallel-worker-manager|worker-manager]], [[components/parallel-task-queue|task-queue]], [[components/parallel-interrupt|interrupt]]
- Caller: [[components/query-executor|qexec_execute_mainblock]]
- Hub: [[components/parallel-query-execute|parallel-query-execute]]
