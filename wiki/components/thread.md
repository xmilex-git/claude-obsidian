---
type: component
parent_module: "[[modules/src|src]]"
path: "src/thread/"
status: stable
purpose: "C++17 threading infrastructure: thread entry context, worker pools, background daemons, and legacy compatibility layer"
key_files:
  - "thread_manager.hpp / thread_manager.cpp (cubthread::manager singleton)"
  - "thread_worker_pool.hpp (abstract worker_pool base)"
  - "thread_worker_pool_impl.hpp (worker_pool_impl<Stats> — concrete)"
  - "thread_entry.hpp (THREAD_ENTRY / cubthread::entry)"
  - "thread_entry_task.hpp (entry_task, entry_manager)"
  - "thread_task.hpp (task<Context>, callable_task<Context>)"
  - "thread_daemon.hpp (cubthread::daemon)"
  - "thread_looper.hpp (cubthread::looper — sleep strategy)"
  - "thread_waiter.hpp (cubthread::waiter — condition variable)"
  - "thread_compat.hpp (THREAD_ENTRY typedef for cross-module use)"
  - "critical_section.c (legacy named mutex — CS)"
  - "critical_section_tracker.hpp (CS acquisition ordering / deadlock detection)"
public_api:
  - "cubthread::get_manager() — access the singleton manager"
  - "cubthread::get_entry() — get current thread's THREAD_ENTRY"
  - "cubthread::manager::create_worker_pool<Res>(pool_size, core_count, ...)"
  - "cubthread::manager::push_task(worker_pool*, entry_task*)"
  - "cubthread::manager::create_daemon(looper, entry_task*, name)"
  - "cubthread::manager::create_daemon_without_entry(looper, task<void>*, name)"
  - "cubthread::initialize(entry*&) / cubthread::finalize()"
  - "thread_create_worker_pool(...) / thread_create_stats_worker_pool(...) — C shim"
  - "REGISTER_CONNECTION / REGISTER_WORKERPOOL / REGISTER_DAEMON macros"
tags:
  - component
  - cubrid
  - thread
  - concurrency
related:
  - "[[modules/src|src]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[components/worker-pool|worker-pool]]"
  - "[[components/entry-task|entry-task]]"
  - "[[components/thread-daemon|thread-daemon]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/lockfree|lockfree]]"
  - "[[components/vacuum|vacuum]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/thread/` — `cubthread` Namespace Overview

The `src/thread/` directory provides the complete threading infrastructure for the CUBRID server engine. All server-side subsystems obtain thread context, dispatch parallel work, and register background daemons through the primitives here.

## Architecture

```
cubthread::manager  (singleton, SERVER_MODE + SA_MODE)
  │
  ├── entry[]          pre-allocated THREAD_ENTRY pool (fixed size at startup)
  │     └── entry_dispatcher  (resource_shared_pool — claim/retire)
  │
  ├── worker_pool[]    registered pools (e.g. "parallel-query", vacuum workers)
  │     └── worker_pool_impl<Stats>
  │           ├── core_impl[]   (one per NUMA-like affinity group)
  │           │     └── worker[]  (actual std::thread wrappers)
  │           └── task queue    (per-core, for high-load buffering)
  │
  └── daemon[]         background daemons (vacuum master, log flusher, etc.)
        ├── looper     (sleep strategy: INF / FIXED / INCREASING / CUSTOM)
        └── waiter     (condition variable wrapper)
```

## Core Abstractions

| Abstraction | Header | Summary |
|-------------|--------|---------|
| `cubthread::entry` | `thread_entry.hpp` | Per-thread context struct — the `thread_p` parameter everywhere |
| `cubthread::manager` | `thread_manager.hpp` | Singleton registry for pools, daemons, and entries |
| `cubthread::worker_pool` | `thread_worker_pool.hpp` | Abstract pool base; concrete is `worker_pool_impl<Stats>` |
| `cubthread::entry_task` | `thread_entry_task.hpp` | `task<entry>` — base class for all pool tasks |
| `cubthread::daemon` | `thread_daemon.hpp` | Background long-running thread with looper |
| `cubthread::looper` | `thread_looper.hpp` | Configurable sleep/wake pattern |
| `cubthread::waiter` | `thread_waiter.hpp` | Condition variable wrapper |

## THREAD_ENTRY — The Universal Context

`cubthread::entry` (aliased `THREAD_ENTRY`) is the most-passed pointer in the codebase. It carries:

- **Identity**: `index`, `type` (`TT_WORKER`, `TT_DAEMON`, `TT_VACUUM_WORKER`, etc.), `tran_index`
- **Memory**: `private_heap_id` (thread-local allocator for `db_private_alloc`)
- **Error context**: `cuberr::context` — thread-local error stack
- **Lock-free slots**: `lf_tran_entry*[THREAD_TS_LAST]` — 11 transaction slots for lock-free structures
- **Locking**: `lockwait`, `lockwait_state` — current lock wait info
- **Page buffer**: `pgbuf_holder_anchor` — page fix tracking
- **Interrupt**: `interrupted`, `shutdown` (atomic) — cooperative cancellation
- **Vacuum**: `vacuum_worker*` pointer (non-NULL for vacuum worker threads)
- **Parallel query**: `m_px_stats`, `m_px_orig_thread_entry` — PX bookkeeping

Entries are pooled: startup allocates `m_max_threads` entries; workers claim one on start and retire it on stop. Construction is cheap; re-initialization is done by `entry_manager`.

## Build Mode Behavior

| Mode | Worker pools | Daemons | Notes |
|------|-------------|---------|-------|
| `SERVER_MODE` | Full `worker_pool_impl` | Fully functional | Default server build |
| `SA_MODE` | NULL (tasks run inline) | Not created (`assert(false)`) | Single-thread embedded |
| `CS_MODE` | N/A — `thread_manager.hpp` excluded | N/A | Client-only |

> [!key-insight] SA_MODE inline execution
> In SA_MODE, `push_task(NULL_pool, task)` executes the task synchronously on the calling thread then calls `task->retire()`. No threads are spawned. This means parallel-query, vacuum, and other pool-based features are effectively serial in SA_MODE.

## Logging Flags

Thread activity is gated by `PRM_ID_THREAD_LOGGING_FLAG` (checked via `cubthread::is_logging_configured`):

| Constant | Hex | Scope |
|----------|-----|-------|
| `LOG_WORKER_POOL_VACUUM` | `0x100` | Vacuum worker pool |
| `LOG_WORKER_POOL_CONNECTIONS` | `0x200` | Connection worker pool |
| `LOG_WORKER_POOL_TRAN_WORKERS` | `0x400` | Transaction workers |
| `LOG_WORKER_POOL_INDEX_BUILDER` | `0x800` | Index build workers |
| `LOG_DAEMON_VACUUM` | `0x10000` | Vacuum daemon |

## Registration Macros

Pools and daemons declare their capacity at static init time via:

```cpp
REGISTER_WORKERPOOL(my_pool, []() { return prm_get_integer_value(PRM_ID_...); });
REGISTER_DAEMON(my_daemon);
REGISTER_CONNECTION(my_conn_pool, getter);
```

`set_max_thread_count_from_config()` sums all registered counts to determine entry array size.

## Consumers

- [[components/parallel-query|parallel-query]] — owns the `"parallel-query"` worker pool; tasks are `entry_task` subclasses dispatched via `get_manager()->push_task()`
- [[components/parallel-worker-manager|parallel-worker-manager]] — wraps the parallel-query pool with per-query reservation semantics
- [[components/vacuum|vacuum]] — vacuum master daemon + up-to-50 vacuum worker pool
- [[components/log-manager|log-manager]] — log flush daemon
- [[components/page-buffer|page-buffer]] — page flusher and checkpoint daemons
- [[components/lockfree|lockfree]] — uses `lf_tran_entry` slots inside `THREAD_ENTRY`

## Sub-Component Pages

- [[components/thread-manager|thread-manager]] — singleton manager: pool registry, entry dispatch, `get_manager()`
- [[components/worker-pool|worker-pool]] — `worker_pool_type`, `execute`, `execute_on_core`, drain
- [[components/entry-task|entry-task]] — `entry_task` abstract base, retire pattern, `entry_manager`
- [[components/thread-daemon|thread-daemon]] — daemon lifecycle, looper strategies, known daemons

## Related

- Source: [[sources/cubrid-src-thread|cubrid-src-thread]]
- [[Memory Management Conventions]] — `db_private_alloc` uses `entry->private_heap_id`
- [[Build Modes (SERVER SA CS)]] — SA_MODE behavior differences
