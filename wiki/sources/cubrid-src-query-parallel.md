---
type: source
title: "CUBRID src/query/parallel/ — Parallel Query Subsystem"
source_type: directory
source_path: ".raw/cubrid/src/query/parallel/"
ingested: 2026-04-23
status: summarized
tags:
  - source
  - cubrid
  - parallel
  - query
related:
  - "[[CUBRID]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/parallel-hash-join|parallel-hash-join]]"
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-query-execute|parallel-query-execute]]"
  - "[[components/parallel-sort|parallel-sort]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# Source: `src/query/parallel/`

CUBRID's parallel query execution subsystem. 16 top-level files + 3 subdirectories (`px_hash_join/`, `px_heap_scan/`, `px_query_execute/`).

## Read scope (this ingest pass)

- All headers (.hpp/.h): `px_parallel.hpp`, `px_callable_task.hpp`, `px_thread_safe_queue.hpp`, `px_worker_manager.hpp`, `px_worker_manager_global.hpp`, `px_interrupt.hpp`, `px_sort.h`
- Representative implementations: `px_parallel.cpp`, `px_worker_manager.cpp`
- Subdirs: header sweep + 1 representative `.cpp` each

## Highest-leverage facts

> [!key-insight] Single global worker pool
> All parallel queries borrow workers from one named `cubthread::worker_pool_type` ("parallel-query") owned by a `worker_manager_global` singleton. Per-query `worker_manager` instances reserve slots via lock-free CAS.

> [!key-insight] Logarithmic auto-degree
> `compute_parallel_degree(type, num_pages, hint_degree)` picks the parallel degree as `floor(log2(num_pages / threshold)) + 2`. Capped by `PRM_ID_PARALLELISM` and core count. Returns 0 to disable on 1-2 core systems.

> [!key-insight] Thread-local errors must be moved
> CUBRID stores error context in thread-local `cuberr::context`. Worker threads MUST explicitly snapshot their error into `err_messages_with_lock` before exit, otherwise the main thread can't see it. `move_top_error_message_to_this()` is the move primitive.

> [!key-insight] Spin-yield wait, not condvar (for the manager)
> `worker_manager::wait_workers()` busy-spins with `std::this_thread::yield()` rather than condvar. Intentional for short-lived parallel bursts. `px_sort` uses condvar instead because sort-run generation is longer-lived.

> [!key-insight] Build-mode-fenced
> All files except `px_parallel.{hpp,cpp}` itself are guarded `#if !defined(SERVER_MODE) && !defined(SA_MODE)` — purely server-side / standalone, never client. See [[Build Modes (SERVER SA CS)]].

## Pages produced

Component pages (under `wiki/components/`):
- [[components/parallel-query|parallel-query]] — overview & architecture
- [[components/parallel-worker-manager|parallel-worker-manager]] — pool lifecycle, reservation
- [[components/parallel-task-queue|parallel-task-queue]] — MPMC queue + callable_task
- [[components/parallel-hash-join|parallel-hash-join]] — `px_hash_join/`
- [[components/parallel-heap-scan|parallel-heap-scan]] — `px_heap_scan/`
- [[components/parallel-query-execute|parallel-query-execute]] — `px_query_execute/`
- [[components/parallel-sort|parallel-sort]] — `px_sort.{h,c}`

## Cross-references touched

- [[components/storage|storage]] — `external_sort.c` is the consumer of [[components/parallel-sort|parallel-sort]]
- [[Query Processing Pipeline]] — parallel paths slot into the XASL executor stage
- [[Memory Management Conventions]] — `memory_wrapper.hpp` last-include rule observed in `px_sort.c`
- [[Code Style Conventions]] — `// *INDENT-OFF*` markers around macros in `px_sort.h`

## Suggested follow-ups

- Flow page: "Parallel hash join request path" — how a hash join in [[components/query|query]] decides parallel vs serial and dispatches.
- Deeper read: `external_sort.c` (storage) — to fully document the sort partition contract.
- ADR candidate: "Why a singleton worker pool with reservation" — design rationale (likely in commit history / discussion).

## Source location

`.raw/cubrid/src/query/parallel/` (symlink → `/Users/song/DEV/cubrid/src/query/parallel/`).
