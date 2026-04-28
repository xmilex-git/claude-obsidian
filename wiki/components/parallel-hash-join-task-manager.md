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
  ├─ [outer loop: get_next_page()]                ← LOCK-FREE (sector bitmap or membuf CAS)
  │    ├─ check has_error / check_interrupt
  │    ├─ skip page if QFILE_GET_TUPLE_COUNT == 0 (empty page)
  │    ├─ handle overflow start page → walk QFILE_GET_OVERFLOW_VPID chain
  │    │     fetch/release continuation pages using m_current_tfile
  │    └─ [inner loop: tuples on page]
  │         ├─ hjoin_fetch_key() — extract hash key
  │         ├─ compute part_id = hash_key % part_cnt (or part_cnt-1 for NULL outer join)
  │         └─ append tuple to temp_part_list_id[part_id]
  │              when temp partition is full → lock part_mutex,
  │              qfile_append_list + qfile_truncate_list (or first-fill copy)
  │
  ├─ qmgr_free_old_page_and_init(page, m_current_tfile)
  └─ flush all temp_part_list_ids to shared partitions (under per-partition mutex)
       success branch: append + truncate, keep LIST_ID descriptor; cleanup branch is a SEPARATE if (not else)
~task_execution_guard                              ← restore thread identity
```

### Page Cursor — Lock-free sector bitmap walk (`get_next_page`)

```
Phase 1: membuf
  if m_membuf_index >= 0:                          ← already the membuf owner
      walk membuf[0..membuf_last] sequentially
      vpid = {NULL_VOLID, m_membuf_index++}
      record m_current_tfile = sector_info->membuf_tfile
      skip pages where TUPLE_COUNT == QFILE_OVERFLOW_TUPLE_COUNT_FLAG
      return page

  if m_sector_index == -1 and sector_info->membuf_tfile != NULL:
      try membuf_claimed.compare_exchange_strong(false → true, acq_rel)
        winner: m_membuf_index = 0 ; re-enter Phase 1
        loser:  fall to Phase 2

Phase 2: sector
  while true:
      while m_current_bitmap != 0:
          bit = __builtin_ctzll(m_current_bitmap)
          m_current_bitmap &= m_current_bitmap - 1   ← clear lowest set bit
          vpid = { m_current_vsid.volid, SECTOR_FIRST_PAGEID(m_current_vsid.sectid) + bit }
          tfile = (QMGR_TEMP_FILE *) sector_info->tfiles[m_sector_index]
          page = qmgr_get_old_page(thread, &vpid, tfile)
          skip overflow continuation pages (TUPLE_COUNT == QFILE_OVERFLOW_TUPLE_COUNT_FLAG)
          record m_current_tfile = tfile
          return page

      sector_index = next_sector_index.fetch_add(1, relaxed)
      if sector_index >= sector_info->sector_cnt:
          return nullptr                             ← end of scan
      m_sector_index = sector_index
      m_current_vsid = sectors[sector_index].vsid
      m_current_bitmap = sectors[sector_index].page_bitmap
```

> [!key-insight] Lock-free split distribution
> Pre-PR #6981 every `get_next_page` call took `HASHJOIN_SHARED_SPLIT_INFO::scan_mutex` to advance a shared `(scan_position, next_vpid)` cursor — wall-clock split throughput was bounded by mutex hold time. Post-PR, membuf has exactly one owner (single CAS at startup) and disk pages are distributed via `next_sector_index.fetch_add(1, relaxed)`; bitmap iteration inside a sector is purely thread-local. The cost is one upfront `qfile_collect_list_sector_info` call on the main thread before the worker fan-out, which itself reuses `file_get_all_data_sectors` already on the data path for parallel heap scan and parallel index build.

> [!key-insight] Per-thread `m_current_tfile` for dependent lists
> When `QFILE_LIST_ID` has a `dependent_list_id` chain (e.g. inner side of a nested-loop join materialised into a secondary list), each list owns its own `QMGR_TEMP_FILE *tfile_vfid`. A single shared base-list tfile pointer is wrong for pages from dependents — `qmgr_free_old_page_and_init` must be called against the correct tfile. PR #6981 records the owning tfile in `m_current_tfile` whenever `get_next_page` returns a page; `execute()` then uses it for both the page release and the entire `QFILE_GET_OVERFLOW_VPID` continuation-page chain (continuation pages are allocated by `qfile_allocate_new_ovf_page` from the same tfile as the start page).

> [!key-insight] Per-thread temp list files reduce lock contention
> Each `split_task` builds up to `part_cnt` in-memory temp list files (`temp_part_list_id[]`) locally. Only when a temp partition fills a memory page does it flush under the per-partition mutex. This amortises lock acquisition across many tuples per page. PR #6981 additionally swapped the post-flush `qfile_destroy_list + free` for `qfile_truncate_list + retain LIST_ID`: the temp partition descriptor is reused for the next batch, and a mid-flush `qfile_append_list` failure can no longer leave a half-freed `LIST_ID` for the cleanup branch to double-free.

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
