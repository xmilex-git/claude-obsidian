---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_sort.{h,c}"
status: active
purpose: "Parallel external sort: split sort run generation across worker threads via macro-based dispatch and condvar coordination"
key_files:
  - "px_sort.h (SORT_EXECUTE_PARALLEL / SORT_WAIT_PARALLEL macros, px_status enum, parallel_type C enum, function prototypes)"
  - "px_sort.c (sort_copy_sort_info — deep copy of SORT_INFO for worker threads)"
  - "external_sort.h / external_sort.c (consumer / hand-off point)"
public_api:
  - "SORT_EXECUTE_PARALLEL(num, px_sort_param, function)"
  - "SORT_WAIT_PARALLEL(num, sort_param, px_sort_param)"
  - "SORT_IS_PARALLEL(t)"
  - "sort_listfile_execute(entry&, SORT_PARAM*)"
  - "sort_copy_sort_param(THREAD_ENTRY*, SORT_PARAM* dest, SORT_PARAM* src, int parallel_num)"
  - "sort_copy_sort_info(THREAD_ENTRY*, SORT_INFO** dest, SORT_INFO* src)"
  - "sort_split_input_temp_file(THREAD_ENTRY*, SORT_PARAM*, SORT_PARAM*, int)"
  - "sort_merge_run_for_parallel(THREAD_ENTRY*, SORT_PARAM*, SORT_PARAM*, int)"
  - "sort_merge_nruns(THREAD_ENTRY*, SORT_PARAM*)"
  - "sort_check_parallelism(THREAD_ENTRY*, SORT_PARAM*)"
  - "sort_start_parallelism(THREAD_ENTRY*, SORT_PARAM* px, SORT_PARAM* src)"
  - "sort_end_parallelism(THREAD_ENTRY*, SORT_PARAM* px, SORT_PARAM* src)"
  - "sort_put_result_for_parallel(entry&, SORT_PARAM*)"
  - "sort_merge_nruns_parallel(entry&, SORT_PARAM*)"
  - "sort_split_last_run(THREAD_ENTRY*, SORT_PARAM*, SORT_PARAM*, int)"
  - "sort_put_result_from_tmpfile(THREAD_ENTRY*, SORT_PARAM*, int)"
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
  - "[[components/external-sort|external-sort]]"
  - "[[components/storage|storage]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_sort` — Parallel External Sort

Adds parallelism to CUBRID's external sort (`src/storage/external_sort.c`). Each worker handles a subset of the input partitions; the main thread waits on a POSIX condvar until all workers report `PX_DONE` or one reports `PX_ERR_FAILED`.

---

## `px_status` Enum — Complete Values

Defined in `px_sort.h`:

```c
enum px_status {
    PX_ERR_FAILED = -1,   // worker encountered an error
    PX_DONE       =  0,   // worker completed successfully
    PX_PROGRESS           // worker still running (= 1, implicit)
};
typedef enum px_status PX_STATUS;
```

| Value | Integer | Meaning |
|-------|---------|---------|
| `PX_ERR_FAILED` | -1 | Worker failed; main thread sets `error = ER_FAILED` but continues waiting |
| `PX_DONE` | 0 | Worker completed successfully |
| `PX_PROGRESS` | 1 | Worker still running; main thread must continue waiting |

---

## C-level `parallel_type` Enum

Also defined in `px_sort.h` (distinct from `parallel_query::parallel_type` in C++):

```c
enum parallel_type {
    PX_SINGLE            = 0,   // no parallelism
    PX_MAIN_IN_PARALLEL  = 1,   // main thread participates in sort
    PX_THREAD_IN_PARALLEL        // dedicated worker threads only
};
typedef enum parallel_type PARALLEL_TYPE;
```

---

## Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `SORT_PX_MERGE_FILES` | `4` | Number of merge files per parallel worker |
| `SORT_MAX_PARALLEL` | `PRM_MAX_PARALLELISM` | Maximum parallel degree for sort |
| `SORT_IS_PARALLEL(t)` | macro | True when `(t)->px_parallel_num > 1` |

---

## Function Inventory

| Function | Signature | Role |
|----------|-----------|------|
| `sort_listfile_execute` | `void(entry&, SORT_PARAM*)` | Entry point for a worker thread — executes one sort partition |
| `sort_copy_sort_param` | `int(THREAD_ENTRY*, SORT_PARAM* dest, SORT_PARAM* src, int n)` | Deep-copies `n` sort params for parallel workers — **implementation lives at `external_sort.c:4344-4471` (next to its consumer), not in `px_sort.c`**. PR #7011 (`cc563c7`) added the implementation; the symbol's prior presence in `px_sort.h` without a matching definition was a pre-merge hot-cache nit, now resolved. |
| `sort_copy_sort_info` | `int(THREAD_ENTRY*, SORT_INFO**, SORT_INFO*)` | Deep-copies `SORT_INFO` including nested `QFILE_SORT_SCAN_ID` and `QFILE_LIST_SCAN_ID` |
| `sort_split_input_temp_file` | `int(THREAD_ENTRY*, SORT_PARAM*, SORT_PARAM*, int)` | Splits the input temp file across `parallel_num` workers |
| `sort_merge_run_for_parallel` | `int(THREAD_ENTRY*, SORT_PARAM*, SORT_PARAM*, int)` | Merges runs across the parallel sort results |
| `sort_merge_nruns` | `int(THREAD_ENTRY*, SORT_PARAM*)` | Merges N runs within a single worker's partition |
| `sort_check_parallelism` | `int(THREAD_ENTRY*, SORT_PARAM*)` | Validates sort params are suitable for parallel execution |
| `sort_start_parallelism` | `int(THREAD_ENTRY*, SORT_PARAM* px, SORT_PARAM* src)` | Sets up parallel context; calls `try_reserve_workers` |
| `sort_end_parallelism` | `int(THREAD_ENTRY*, SORT_PARAM* px, SORT_PARAM* src)` | Merges parallel results; calls `release_workers` |
| `sort_put_result_for_parallel` | `void(entry&, SORT_PARAM*)` | Parallel worker: writes sort run output to temp file |
| `sort_merge_nruns_parallel` | `void(entry&, SORT_PARAM*)` | Parallel worker: merge N runs for one partition |
| `sort_split_last_run` | `void(THREAD_ENTRY*, SORT_PARAM*, SORT_PARAM*, int)` | Redistributes the last run across workers for balance |
| `sort_put_result_from_tmpfile` | `int(THREAD_ENTRY*, SORT_PARAM*, int start_pagenum)` | Reads results back from temp file into the main sort stream |

---

## Where It Sits — `external_sort.c` Hand-Off Points

```
external_sort.c (src/storage/)
        │
        ├─ sort_check_parallelism()      check degree > 1 and workers available
        ├─ sort_start_parallelism()      deep-copy params, reserve workers
        │
        ├─ SORT_EXECUTE_PARALLEL(...)    dispatch N callable_tasks to pool
        ├─ SORT_WAIT_PARALLEL(...)       condvar wait for all PX_DONE/PX_ERR_FAILED
        │
        └─ sort_end_parallelism()        merge results, release workers
```

`px_sort.h` is `#include`d by `external_sort.c` directly — the sort macros expand inline in the storage-layer file.

---

## Macro Expansion Trace — `SORT_EXECUTE_PARALLEL`

**Source (from `px_sort.h`):**

```c
#define SORT_EXECUTE_PARALLEL(num, px_sort_param, function)      \
    do {                                                         \
        for (int i = 0; i < num; i++) {                          \
            parallel_query::callable_task *task =                \
               new parallel_query::callable_task(                \
                  sort_param->px_worker_manager,                 \
                  std::bind(function,                            \
                            std::placeholders::_1,               \
                            &px_sort_param[i]));                 \
            sort_param->px_worker_manager->push_task(task);      \
        }                                                        \
    } while (0)
```

**Expanded for `num = 4`, `function = sort_listfile_execute`:**

```cpp
do {
    for (int i = 0; i < 4; i++) {
        parallel_query::callable_task *task =
            new parallel_query::callable_task(
                sort_param->px_worker_manager,
                std::bind(sort_listfile_execute,
                          std::placeholders::_1,   // cubthread::entry& — filled by worker
                          &px_sort_param[i]));     // per-partition SORT_PARAM*
        sort_param->px_worker_manager->push_task(task);
    }
} while (0);
```

Each `callable_task` captures a pointer to its partition's `SORT_PARAM` via `std::bind`. The `std::placeholders::_1` is the `cubthread::entry&` supplied by the pool worker at call time. Default `delete_on_retire = true` is used — tasks heap-allocate themselves and self-destruct in `retire()`.

---

## Macro Expansion Trace — `SORT_WAIT_PARALLEL`

**Source (from `px_sort.h`):**

```c
#define SORT_WAIT_PARALLEL(parallel_num, sort_param, px_sort_param)    \
    do {                                                                \
        pthread_mutex_lock(sort_param->px_mtx);                        \
        while (1) {                                                     \
            int done = true;                                            \
            for (int i = 0; i < parallel_num; i++) {                   \
                if (px_sort_param[i].px_status == PX_PROGRESS) {       \
                    done = false;                                       \
                    break;                                              \
                } else if (px_sort_param[i].px_status == PX_ERR_FAILED) { \
                    error = ER_FAILED;                                  \
                }                                                       \
            }                                                           \
            if (done) break;                                            \
            pthread_cond_wait(sort_param->complete_cond, sort_param->px_mtx); \
        }                                                               \
        pthread_mutex_unlock(sort_param->px_mtx);                      \
        sort_param->px_worker_manager->wait_workers();                  \
    } while (0)
```

**Step-by-step behaviour:**

1. Acquire `px_mtx` (POSIX mutex embedded in `SORT_PARAM`).
2. Loop over all `px_sort_param[i].px_status` values:
   - If any is `PX_PROGRESS` → `done = false`, call `pthread_cond_wait` (releases mutex, blocks).
   - If any is `PX_ERR_FAILED` → set `error = ER_FAILED` but continue — do NOT break out early; wait for all workers.
   - If all are `PX_DONE` or `PX_ERR_FAILED` → `done = true`, break.
3. Release `px_mtx`.
4. Call `worker_manager::wait_workers()` — spin until `m_active_tasks == 0`.

The macro requires a local variable `error` in scope at the call site.

> [!key-insight] Condvar here vs busy-spin in `wait_workers`
> `SORT_WAIT_PARALLEL` uses `pthread_cond_wait` because sort workers signal completion status per-partition and may take significant time (disk I/O during external merge). Blocking the main thread with a condvar is the right choice here — it allows the OS scheduler to use the main thread's CPU for other work. By contrast, `worker_manager::wait_workers()` uses a yield-spin because it is a short, final drain waiting only for the CUBRID pool to call `retire()` after `execute()` returns — a gap measured in microseconds.
>
> The two-phase wait (condvar → yield-spin) reflects this: `SORT_WAIT_PARALLEL` waits for logical completion (all partitions done), then `wait_workers()` waits for the task infrastructure to finish housekeeping.

---

## `sort_copy_sort_info` — Deep Copy

The only function implemented in `px_sort.c` (SERVER_MODE only):

```c
int sort_copy_sort_info(THREAD_ENTRY *thread_p, SORT_INFO **dest_sort_info, SORT_INFO *src_sort_info) {
    // Allocates: SORT_INFO → QFILE_SORT_SCAN_ID → QFILE_LIST_SCAN_ID
    // All three levels via db_private_alloc (thread-local private heap)
    // memcpy at each level
    // Sets sort_info->output_file = NULL, sort_info->input_file = NULL
    //   (each worker gets its own output file, nulling prevents aliasing)
    // On OOM at any level: unwinds with db_private_free_and_init
}
```

This deep copy is required because `SORT_INFO` contains pointer-to-pointer scan state that must be unique per worker thread — sharing scan state would cause races.

---

## `result_run` Struct

```c
typedef struct result_run RESULT_RUN;
struct result_run {
    VFID temp_file;   // virtual file ID of the worker's sort-run output
    int  num_pages;   // number of pages written
};
```

Used to track per-worker sort run output before the merge phase.

---

## Build-Mode Guard

```c
#if !defined(SERVER_MODE) && !defined(SA_MODE)
#error Belongs to server module
#endif
```

`px_sort.h` is a C header (`.h`, not `.hpp`). `px_sort.c` is compiled as C++17 (all `.c` files in this project use `c_to_cpp.sh`).

---

## Macro Formatting Notes

Both macros use `// *INDENT-OFF*` / `// *INDENT-ON*` guards around `SORT_EXECUTE_PARALLEL` in the header. This bypasses `indent`/`astyle` formatting because the multi-line backslash continuation would be mangled by the code formatter. `SORT_WAIT_PARALLEL` does not have these guards (it uses C-style `/* */`-compatible macros with consistent indentation that the formatter handles).

---

## Execution Path

```
external_sort.c
  ├─ sort_check_parallelism(thread_p, sort_param)
  │      └─ verifies px_parallel_num > 1 and worker reservation possible
  │
  ├─ sort_start_parallelism(thread_p, px_sort_param, sort_param)
  │      ├─ sort_copy_sort_param() × parallel_num
  │      ├─ sort_copy_sort_info() × parallel_num
  │      ├─ sort_split_input_temp_file() — divide input across workers
  │      └─ worker_manager::try_reserve_workers(parallel_num)
  │
  ├─ SORT_EXECUTE_PARALLEL(num, px_sort_param, sort_listfile_execute)
  │      └─ new callable_task(wm, bind(fn, _1, &px[i])) × num
  │         wm->push_task(task) × num
  │
  ├─ SORT_WAIT_PARALLEL(num, sort_param, px_sort_param)
  │      ├─ pthread_cond_wait loop on PX_PROGRESS
  │      └─ wm->wait_workers() final drain
  │
  └─ sort_end_parallelism(thread_p, px_sort_param, sort_param)
         ├─ sort_merge_run_for_parallel()
         └─ worker_manager::release_workers()
```

---

## `SORT_INDEX_LEAF` dispatch (PR #7011)

> [!update] PR #7011 (merge `cc563c7f`) — parallel CREATE INDEX wired through this layer
> `sort_check_parallelism`, `sort_start_parallelism`, `sort_end_parallelism`, and `sort_return_used_resources` all gain `SORT_INDEX_LEAF` arms.

- `sort_check_parallelism` for `SORT_INDEX_LEAF`: returns `1` (single-process) if `n_classes > 1`; otherwise calls `file_get_num_data_sectors(thread_p, &hfid->vfid, ...)` and decides parallelism based on sector count vs threshold.
- `sort_start_parallelism` for `SORT_INDEX_LEAF` (`external_sort.c:5248-5310`): per-worker `malloc(SORT_ARGS)` + `memcpy(sort_param->get_arg)`, override `get_fn = &btree_sort_get_next_parallel`, NULL the inherited filter pointers, then `file_get_all_data_sectors` + `ftab_set::split(parallel_num)` populates per-worker `ftab_sets` (one entry per `n_classes`, possibly empty).
- `sort_end_parallelism` for `SORT_INDEX_LEAF`: dispatches `sort_merge_run_for_parallel_index_leaf_build` (the index-leaf log₄ tree-merge) instead of the ORDER_BY merge.
- `sort_return_used_resources` checks `if (sort_param->get_arg != NULL)` before freeing per-worker `SORT_ARGS`, and uses explicit `~vector()` + `free_and_init` for `ftab_sets` (allocated via `malloc` + placement-`new`).

See [[components/external-sort#index-leaf-parallel-build-sort_index_leaf]] and [[components/btree#parallel-index-build-sort_index_leaf]] for the call-site details.

## Constraints

- `SORT_WAIT_PARALLEL` requires a local `int error` in scope — it assigns `error = ER_FAILED` directly.
- `sort_copy_sort_info` is conditionally compiled for `SERVER_MODE` only (not `SA_MODE`) — the `#if defined(SERVER_MODE)` guard wraps the function in `.c`.
- All `sort_copy_*` allocations use `db_private_alloc` bound to `thread_p` — must be freed with `db_private_free_and_init` on the same thread context.
- `POSIX` mutexes (`pthread_mutex_t`) and condvars (`pthread_cond_t`) are used rather than C++ `std::mutex` / `std::condition_variable` — consistent with `SORT_PARAM` being a C struct with C-compatible synchronization primitives.

---

## Related

- [[components/parallel-query|parallel-query]] — overview and degree selection
- [[components/parallel-worker-manager|parallel-worker-manager]] — the pool used
- [[components/parallel-task-queue|parallel-task-queue]] — `callable_task` dispatched by the macros
- [[components/external-sort|external-sort]] — the consumer calling these macros (`src/storage/external_sort.c`)
- [[components/storage|storage]] — storage layer overview
- [[Memory Management Conventions]] — `db_private_alloc`, `db_private_free_and_init`
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
