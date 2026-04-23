---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_sort.{h,c}"
status: developing
purpose: "Parallel external sort: split sort run generation across worker threads"
key_files:
  - "px_sort.h (SORT_EXECUTE_PARALLEL / SORT_WAIT_PARALLEL macros, px_status enum)"
  - "px_sort.c (callable bindings + condvar coordination)"
  - "external_sort.h (consumer — see src/storage)"
public_api:
  - "SORT_EXECUTE_PARALLEL(num, px_sort_param, function) — dispatch N parallel sort sub-tasks"
  - "SORT_WAIT_PARALLEL(num, sort_param, px_sort_param) — wait for completion / error reporting"
  - "SORT_IS_PARALLEL(t) — predicate (px_parallel_num > 1)"
public_api_constants:
  - "SORT_PX_MERGE_FILES = 4"
  - "SORT_MAX_PARALLEL = PRM_MAX_PARALLELISM"
tags:
  - component
  - cubrid
  - parallel
  - query
  - sort
related:
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/storage|storage]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_sort` — Parallel External Sort

Adds parallelism to CUBRID's external sort (`src/storage/external_sort.c`). Each worker handles a subset of the input partitions; the main thread waits on a condvar until all workers report `PX_DONE` or one reports `PX_ERR_FAILED`.

## Where it sits

```
external_sort.c (storage layer)
        │
        │  uses
        ▼
SORT_EXECUTE_PARALLEL(...)   ← px_sort.h macro
        │
        ▼
parallel_query::callable_task ──► worker_manager ──► thread pool
```

## Macro expansion

`SORT_EXECUTE_PARALLEL` constructs N `callable_task` instances, each binding a per-partition function via `std::bind`, then pushes them to the per-query `worker_manager`. The actual work happens inside the lambda the macro callsite supplied (typically a sort-run generator).

`SORT_WAIT_PARALLEL` enters a condvar loop guarded by `sort_param->px_mtx`. It cycles through `px_sort_param[]` checking `px_status`:
- Any `PX_PROGRESS` → wait again
- Any `PX_ERR_FAILED` → set `error = ER_FAILED`, but still loop until all done
- All `PX_DONE` → break, then call `worker_manager->wait_workers()` to drain the pool

## `px_status` enum

States a worker can report (defined in `px_sort.h`): `PX_PROGRESS`, `PX_DONE`, `PX_ERR_FAILED` (full enum in header — read for exact values).

## Build mode guard

```cpp
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Belongs to server module
#endif
```

Server-side and standalone only — see [[Build Modes (SERVER SA CS)]].

## Includes

`px_sort.c` follows the project [[Memory Management Conventions|memory_wrapper.hpp last-include rule]] (visible at the bottom of the includes block).

## Notes

- Macros use `// *INDENT-OFF*` / `// *INDENT-ON*` to bypass [[Code Style Conventions|indent/astyle]] formatting because the multi-line backslash style breaks the formatter.
- `pthread_mutex_lock` + `pthread_cond_wait` rather than C++ `std::mutex` / `std::condition_variable` — consistent with engine-wide preference for POSIX primitives.

## Related

- [[components/parallel-query|parallel-query]] (overview)
- [[components/parallel-worker-manager|parallel-worker-manager]] (the pool used)
- [[components/parallel-task-queue|parallel-task-queue]] (`callable_task`)
- [[components/storage|storage]] (consumer: `external_sort.c`)
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
