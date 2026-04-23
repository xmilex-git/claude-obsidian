---
type: source
title: "CUBRID src/thread/ — Worker Pools & Daemon Threads"
source_path: "src/thread/"
ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - thread
  - concurrency
related:
  - "[[components/thread|thread]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[components/worker-pool|worker-pool]]"
  - "[[components/entry-task|entry-task]]"
  - "[[components/thread-daemon|thread-daemon]]"
  - "[[components/parallel-query|parallel-query]]"
---

# Source: `src/thread/` — Worker Pools & Daemon Threads

Modern C++17 threading infrastructure for the CUBRID server engine. This directory owns all thread lifecycle management, worker pool dispatch, and background daemon threads.

## Key Files Read

| File | Role |
|------|------|
| `AGENTS.md` | Directory-level architecture guide |
| `thread_manager.hpp` / `thread_manager.cpp` | `cubthread::manager` — singleton pool registry |
| `thread_worker_pool.hpp` | Abstract `worker_pool` base + `core` + `worker` nested hierarchy |
| `thread_worker_pool_impl.hpp` | `worker_pool_impl<Stats>` — concrete templated implementation |
| `thread_entry.hpp` | `cubthread::entry` (`THREAD_ENTRY`) — per-thread context struct |
| `thread_entry_task.hpp` | `entry_task`, `entry_manager`, `daemon_entry_manager` |
| `thread_task.hpp` | `task<Context>`, `callable_task<Context>` generics |
| `thread_daemon.hpp` | `cubthread::daemon` — background daemon class |
| `thread_looper.hpp` | `cubthread::looper` — sleep/wake strategy |
| `thread_waiter.hpp` | `cubthread::waiter` — condition variable wrapper |
| `thread_compat.hpp` | `THREAD_ENTRY` typedef, cross-module compatibility |

## Key Findings

1. **SA_MODE is single-thread**: `create_worker_pool` returns NULL in SA_MODE; `push_task` with NULL pool executes the task immediately on the calling thread.
2. **Entry pooling**: `THREAD_ENTRY` structs are pre-allocated in a fixed-size array at startup; `claim_entry`/`retire_entry` recycle them cheaply rather than constructing/destructing.
3. **Two daemon flavors**: `create_daemon` (with `THREAD_ENTRY` context) vs `create_daemon_without_entry` (context-less `task<void>`).
4. **`worker_pool_type` alias**: In `SERVER_MODE` resolves to `worker_pool_impl<false>`; in SA_MODE to the abstract `worker_pool` base (which is never actually used — pool is NULL).
5. **`set_max_thread_count_from_config`**: Sums connection + worker-pool + daemon counts from `count_registry` static accumulators (populated by `REGISTER_*` macros at static init time).
6. **Thread-local entry pointer**: `tl_Entry_p` is a `thread_local entry*`; `cubthread::get_entry()` asserts non-null and returns it.
7. **`system_core_count()`**: Exposed via `os::resources::cpu::effective()` (in `src/base/resources.hpp`), consumed by `parallel_query::compute_parallel_degree()`.

## Pages Created

- [[components/thread]] — component hub
- [[components/thread-manager]] — manager singleton
- [[components/worker-pool]] — worker_pool and worker_pool_impl
- [[components/entry-task]] — entry_task abstract base and retire pattern
- [[components/thread-daemon]] — daemon lifecycle and looper patterns
