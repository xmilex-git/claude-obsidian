---
status: developing
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_hash_join/px_hash_join_task_manager.hpp"
path_impl: "src/query/parallel/px_hash_join/px_hash_join_task_manager.cpp"
tags:
  - component
  - cubrid
  - parallel
  - query
  - hash-join
related:
  - "[[components/parallel-hash-join|parallel-hash-join]]"
  - "[[components/parallel-hash-join-spawn-manager|parallel-hash-join-spawn-manager]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/mvcc|mvcc]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_hash_join_task_manager` — Split and Join Task Inner Logic

Provides five classes that handle the inner execution of parallel hash join workers: the local `task_manager` coordinator, the `task_execution_guard` RAII context setup, and three task classes (`base_task`, `split_task`, `join_task`).

## Class / Function Inventory

| Class | Role |
|-------|------|
| `task_manager` | Local (stack) coordinator — tracks active task count, owns CV + mutex, propagates errors |
| `task_execution_guard` | RAII helper — copies main thread identity into worker thread, manages resource tracks |
| `base_task` | Abstract base (extends `cubthread::entry_task`) — holds `task_manager&`, `HASHJOIN_MANAGER*`, task index |
| `split_task` | Splits input pages into hash partitions |
| `join_task` | Executes build+probe for assigned `HASHJOIN_CONTEXT` objects |

### `task_manager` Methods

| Method | Description |
|--------|-------------|
| `task_manager(worker_manager*, main_thread_ref)` | Ctor; `m_active_tasks = 0`, `m_has_error = false` |
| `push_task(base_task*)` | Increments `m_active_tasks`, calls `worker_manager->push_task` |
| `end_task()` | Decrements `m_active_tasks`, calls `worker_manager->pop_task()`; notifies CV when 0 |
| `join()` | CV-waits until `m_active_tasks == 0`, then calls `worker_manager::wait_workers()` |
| `has_error()` | Returns `m_has_error.load(acquire)` |
| `handle_error(thread_ref)` | Atomically sets `m_has_error`, swaps error context to main thread, calls `notify_stop()`, sets tran interrupt |
| `notify_stop()` | Notifies all waiters on CV (wakes up any sleeping `join()`) |
| `check_interrupt(thread_ref)` | Polls `logtb_is_interrupted_tran`; calls `handle_error` on interrupt |
| `clear_interrupt(thread_ref)` | Drains the interrupt flag after error recovery |

### `split_task` Methods

| Method | Description |
|--------|-------------|
| `split_task(task_manager, manager, split_info, shared_info, index)` | Ctor; asserts `fetch_info != nullptr` and `part_mutexes != nullptr`. Per-thread iteration state initialised: `m_membuf_index = -1`, `m_sector_index = -1`, `m_current_bitmap = 0`, `m_current_vsid = VSID_INITIALIZER`, `m_current_tfile = nullptr`. |
| `execute(thread_ref)` | Main work loop: allocates temp partitions per thread, iterates pages via `get_next_page`, hashes tuples to partitions. All page-release calls and overflow-chain page fetches use the recorded `m_current_tfile`, never `list_id->tfile_vfid`. |
| `get_next_page(thread_ref)` | **Lock-free** two-phase distribution: Phase 1 = membuf CAS-claim (one owner walks all membuf pages); Phase 2 = atomic sector counter (`next_sector_index.fetch_add(1)`) + per-thread `__builtin_ctzll` bitmap walk. Records `m_current_tfile` for use by `execute()`. |
| `retire()` | Calls `task_manager.end_task()`, then `delete this` |

> [!update] PR #6981 (merge `0be6cdf6`) — `split_task` page distribution rewritten lock-free
> The previous design had every worker take `HASHJOIN_SHARED_SPLIT_INFO::scan_mutex` and walk a shared `(scan_position, next_vpid)` cursor one page at a time. PR #6981 replaced the mutex + cursor with sector pre-split via `qfile_collect_list_sector_info` (see [[components/list-file]]) and two atomics (`membuf_claimed`, `next_sector_index`). The five new per-thread `m_*` fields and the `QMGR_TEMP_FILE *` forward-declaration in the hpp header are the visible surface of that change.

### `join_task` Methods

| Method | Description |
|--------|-------------|
| `join_task(task_manager, manager, contexts, shared_info, index)` | Ctor |
| `execute(thread_ref)` | Gets `spawn_manager`, loops: `get_next_context()`, spawns XASL structs, calls `hjoin_execute()` |
| `get_next_context()` | Mutex-protected context index walking `HASHJOIN_SHARED_JOIN_INFO::scan_position` state machine |
| `retire()` | Calls `task_manager.end_task()`, then `delete this` |

## Execution Path — `split_task::execute`

```
task_execution_guard(thread_ref, task_manager)   ← borrow main thread identity
  │
  ├─ alloc temp_part_list_id[part_cnt]            ← per-thread local partitions
  ├─ alloc temp_key (HASH_SCAN_KEY)
  │
  ├─ [outer loop: get_next_page()]                ← lock shared scan_mutex per page
  │    ├─ check has_error / check_interrupt
  │    ├─ handle overflow pages (multi-page tuples)
  │    └─ [inner loop: tuples on page]
  │         ├─ hjoin_fetch_key() — extract hash key
  │         ├─ compute part_id = hash_key % part_cnt (or part_cnt-1 for NULL outer join)
  │         └─ append tuple to temp_part_list_id[part_id]
  │              when temp partition is full → lock part_mutex, merge into shared part_list_id
  │
  └─ flush all temp_part_list_ids to shared partitions (under per-partition mutex)
~task_execution_guard                              ← restore thread identity
```

### Page Cursor State Machine (`get_next_page`)

```
S_BEFORE → (first call, VPID null) → fetch list_id->first_vpid → S_ON / S_AFTER
S_ON     → fetch shared next_vpid → advance → S_AFTER when no more pages
S_AFTER  → return nullptr (end of scan)
```

> [!key-insight] Per-thread temp list files reduce lock contention
> Each `split_task` builds up to `part_cnt` in-memory temp list files (`temp_part_list_id[]`) locally. Only when a temp partition fills a memory page does it flush under the per-partition mutex. This amortises lock acquisition across many tuples per page.

## Execution Path — `join_task::execute`

```
task_execution_guard(thread_ref, task_manager)
  │
  ├─ spawn_manager::get_instance(thread_ref)
  │
  ├─ [loop: get_next_context()]                   ← lock shared scan_mutex per context
  │    ├─ check has_error / check_interrupt
  │    ├─ context->val_descr = spawn_manager->get_val_descr(manager->val_descr)
  │    ├─ context->during_join_pred = spawn_manager->get_during_join_pred(...)
  │    ├─ context->outer.regu_list_pred = spawn_manager->get_outer_regu_list_pred(...)
  │    ├─ context->inner.regu_list_pred = spawn_manager->get_inner_regu_list_pred(...)
  │    ├─ hjoin_execute(thread_ref, manager, context)   ← build hash table + probe
  │    └─ zero out context pointers (not freed; owned by spawn_manager)
  │
  └─ spawn_manager::destroy_instance()           ← TLS cleanup
~task_execution_guard
```

### Context Cursor State Machine (`get_next_context`)

Same three-state machine as `get_next_page` but walks `m_shared_info->next_index` from 0 to `manager->context_cnt`.

## `task_execution_guard` — Thread Identity Borrowing

```cpp
// On construction:
thread_ref.m_px_orig_thread_entry = &main_thread_ref;
thread_ref.conn_entry  = main_thread_ref.conn_entry;
thread_ref.tran_index  = main_thread_ref.tran_index;   // ← joins MVCC snapshot
thread_ref.on_trace    = main_thread_ref.on_trace;
thread_ref.push_resource_tracks();

// On destruction:
thread_ref.conn_entry = nullptr;
thread_ref.on_trace   = false;
thread_ref.pop_resource_tracks();
```

> [!key-insight] Workers join the calling transaction's MVCC snapshot via `tran_index`
> MVCC visibility (`mvcc_satisfies_snapshot`) uses `tran_index` to look up the snapshot. By copying `tran_index` from the main thread, all worker threads see exactly the same snapshot as the main query thread — no per-worker transaction registration needed.

> [!warning] `conn_entry` must not be null
> The guard asserts `conn_entry != nullptr` and `tran_index != NULL_TRAN_INDEX`. If the main thread's connection drops between `build_partitions` and `execute_partitions`, workers would hit an assertion failure. The caller is responsible for ensuring the session remains alive for the duration of both phases.

## Error Propagation

```
worker detects error
  │
  └─ task_manager::handle_error(thread_ref)
       ├─ m_has_error.exchange(true) — first writer wins
       ├─ swap worker error context into main_error_context
       ├─ notify_stop()           ← wakes sleeping join()
       └─ logtb_set_tran_index_interrupt → signals other workers via interrupt flag

main thread (after join()):
  has_error() == true
  clear_interrupt(thread_ref)   ← drains logtb interrupt flag
  return er_errid()             ← error is now in main thread's context
```

## Constraints

- **Memory**: tasks are `new`-allocated and self-delete in `retire()`. Per-task temp allocations use `db_private_alloc` on the worker's `thread_ref`.
- **Threading**: three separate mutexes govern access: `scan_mutex` (page/context cursor), `part_mutexes[i]` (per-partition list), `stats_mutex` (trace time ranges).
- **Interrupt**: `check_interrupt` polls once per page (split) or per context (join). Long-running `hjoin_execute` calls are not interrupted mid-execution.
- **Tracing**: each task sets `thread_ref.m_px_stats` / `m_uses_px_stats` during execution. Join task accumulates `total_build_time` and `total_probe_time` for min/max range stats.
- **Build mode**: active in `SERVER_MODE` and `SA_MODE`.

## Lifecycle

```
1. task_manager constructed (stack, inside build_partitions or execute_partitions)
2. push_task(new split_task / join_task) — m_active_tasks incremented
3. worker calls execute() — task_execution_guard sets up identity
4. worker calls retire() — end_task() decrements m_active_tasks, notifies CV
5. task_manager::join() — CV-wait until m_active_tasks == 0
6. worker_manager::wait_workers() — ensures all threads have returned from execute()
7. task_manager destroyed (stack unwind)
```

## Related

- [[components/parallel-hash-join|parallel-hash-join]] — parent hub and execution phases
- [[components/parallel-hash-join-spawn-manager|parallel-hash-join-spawn-manager]] — `join_task` uses this for XASL structure cloning
- [[components/parallel-worker-manager|parallel-worker-manager]] — `push_task` / `pop_task` / `wait_workers` API
- [[components/mvcc|mvcc]] — MVCC snapshot accessed via borrowed `tran_index`
- [[Memory Management Conventions]] — `db_private_alloc` per-thread allocation model
