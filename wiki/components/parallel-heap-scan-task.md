---
status: developing
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_heap_scan/px_heap_scan_task.hpp"
path_impl: "src/query/parallel/px_heap_scan/px_heap_scan_task.cpp"
tags:
  - component
  - cubrid
  - parallel
  - query
  - heap-scan
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]]"
  - "[[components/parallel-heap-scan-result-handler|parallel-heap-scan-result-handler]]"
  - "[[components/parallel-heap-scan-slot-iterator|parallel-heap-scan-slot-iterator]]"
  - "[[components/parallel-heap-scan-join-info|parallel-heap-scan-join-info]]"
  - "[[components/xasl|xasl]]"
  - "[[components/mvcc|mvcc]]"
  - "[[components/btree|btree]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan_task` — Per-Worker XASL Clone Task

`task<RESULT_TYPE>` is the unit of work dispatched to each worker thread. It extends `cubthread::entry_task`, clones the XASL tree from the cache (or by re-unpacking the stream), opens all necessary scans, and runs the main scan loop feeding results to `result_handler<T>`.

## Class / Function Inventory

| Member | Kind | Description |
|--------|------|-------------|
| `task<result_type>` | class template | Extends `cubthread::entry_task` |
| `task(parent_thread_p, query_entry, result_handler, input_handler, interrupt, err_messages, vd, trace_handler, worker_manager, xasl_id, hfid, cls_oid, is_fixed, is_grouped, uses_xasl_clone, orig_xasl, join_info)` | ctor | All-fields constructor; stores pointers, nulls out m_xasl / m_scan_id / m_vd |
| `~task()` | dtor | Empty; cleanup happens in `finalize()` |
| `execute(thread_ref)` | override | Calls `initialize → loop → finalize`; signals interrupt on init error |
| `retire()` | override | `worker_manager->pop_task(); delete this;` |
| `initialize(thread_ref)` | private | XASL clone, scan open, slot_iterator init, input_handler init, write_initialize |
| `finalize(thread_ref)` | private | Trace merge, write_finalize, input finalize, slot finalize, XASL clear, xcache retire |
| `clone_xasl(thread_ref)` | private | Either `xcache_find_xasl_id_for_execute` or `stx_map_stream_to_xasl`; allocates `xasl_state` |
| `handle_result(thread_ref)` | private | (called from loop) delegates to result_handler write |
| `loop(thread_ref)` | private | Main scan loop: get VPID → set_page → slot iteration → write/join |

### Key Fields

| Field | Type | Role |
|-------|------|------|
| `m_parent_thread_p` | `THREAD_ENTRY*` | Main thread (for identity borrowing + xcache mutex) |
| `m_query_entry` | `QMGR_QUERY_ENTRY*` | Holds XASL SHA-1 id for cache lookup |
| `m_xasl_cache_entry` | `XASL_CACHE_ENTRY*` | Cache entry pinned during execution |
| `m_xasl_clone` | `XASL_CLONE` | {xasl, unpack_info} pair from cache |
| `m_xasl_tree` | `XASL_NODE*` | Root of unpacked tree (non-cache path) |
| `m_xasl_unpack_info` | `XASL_UNPACK_INFO*` | Unpack info (freed on non-cache path) |
| `m_xasl_id` | `int` | `xasl->header.id` of the target node |
| `m_xasl` | `XASL_NODE*` | The specific scan node (found via `xasl_find_by_id`) |
| `m_scan_id` | `SCAN_ID*` | `&m_xasl->spec_list->s_id` |
| `m_slot_iterator` | `slot_iterator` | Per-page slot walker (embedded, not pointer) |
| `m_result_handler` | `result_handler<result_type>*` | Shared result collector |
| `m_input_handler` | `input_handler*` | Shared page distributor |
| `m_join_info` | `join_info*` | Shared join scan-state (for NL joins) |
| `m_uses_xasl_clone` | `bool` | Clone from cache vs. re-unpack from stream |
| `m_scan_func_ptr` | `UINTPTR*` | Array of scan function pointers for multi-level scans |
| `m_start_tick` | `TSC_TICKS` | Trace start time |

## Execution Path

### `execute(thread_ref)`

```
execute(thread_ref)
  ├─ thread_ref.m_px_orig_thread_entry = m_parent_thread_p
  ├─ thread_ref.conn_entry  = m_parent_thread_p->conn_entry
  ├─ thread_ref.tran_index  = m_parent_thread_p->tran_index
  ├─ thread_ref.on_trace    = m_parent_thread_p->on_trace
  │
  ├─ initialize(thread_ref)   [may signal interrupt on failure]
  ├─ loop(thread_ref)
  └─ finalize(thread_ref)
```

### `clone_xasl(thread_ref)`

```
if m_uses_xasl_clone:
  lock main_thread.m_px_lock_mutex
  xcache_find_xasl_id_for_execute(&query_entry->xasl_id, &m_xasl_cache_entry, &m_xasl_clone)
  m_xasl = xasl_find_by_id(m_xasl_clone.xasl, m_xasl_id)
  unlock

else:
  lock main_thread.m_px_lock_mutex
  stx_map_stream_to_xasl(main_thread.xasl_unpack_info_ptr->packed_xasl, ...)
  m_xasl = xasl_find_by_id(m_xasl_tree, m_xasl_id)
  unlock

alloc xasl_state; copy vd from m_orig_vd; clone dbval_ptr values
m_vd = &m_xasl_state->vd
m_scan_id = &m_xasl->spec_list->s_id
```

> [!key-insight] XASL cache access is mutex-protected
> `xcache_find_xasl_id_for_execute` and `stx_map_stream_to_xasl` both run under `main_thread_p->m_px_lock_mutex`. This allows multiple workers to safely clone from the same XASL cache entry concurrently. The lock is released before any actual scan work begins.

### `initialize(thread_ref)` — Scan Opening

```
clone_xasl(thread_ref)

Level 0 (primary heap scan):
  scan_open_heap_scan(&thread_ref, m_scan_id, ..., cls_regu_list_pred, where_pred, ...)
  scan_start_scan(&thread_ref, m_scan_id)

Level 1+ (scan_ptr chain for NL joins — MERGEABLE_LIST / BUILDVALUE_OPT only):
  for each xptr = m_xasl->scan_ptr:
    get scan_info from join_info->get_scan_info(xptr->header.id)
    if TARGET_LIST:
      scan_open_list_scan (using scan_info.list_id from join_info)
    if TARGET_CLASS + ACCESS_METHOD_SEQUENTIAL:
      scan_open_heap_scan (with pruned partition OID + HFID)
    if TARGET_CLASS + ACCESS_METHOD_INDEX:
      scan_open_index_scan (with scan_info.btid)
    scan_start_scan()
    new_memoize_storage(xptr)  ← subquery memoisation

m_slot_iterator.initialize()
m_input_handler->initialize()
m_result_handler->write_initialize()
```

> [!key-insight] Workers open ALL scans in the scan_ptr chain
> For MERGEABLE_LIST and BUILDVALUE_OPT, `task::initialize` opens not just the primary heap scan but every `scan_ptr` sibling — including index scans and list scans needed for NL join inner sides. XASL_SNAPSHOT workers only open the primary heap scan.

> [!update] PR #7049 (`65d6915`, 2026-04-27)
> Renamed `COUNT_DISTINCT` to `BUILDVALUE_OPT` throughout (same enum value 0x3, same `if constexpr` semantics). Also adds a 4-line `er_errid()` check after `write_initialize`: the new failure modes inside `write_initialize<BUILDVALUE_OPT>` (alloc/qfile_open failures) now propagate as `ER_OUT_OF_VIRTUAL_MEMORY` + interrupt code, and this check turns them into a non-zero return from `initialize`. See [[prs/PR-7049-parallel-buildvalue-heap]].

### `loop(thread_ref)`

```
while !stop:
  check m_interrupt
  check logtb_is_interrupted_tran → USER_INTERRUPTED

  scan_code = m_input_handler->get_next_vpid_with_fix(&vpid)  → S_END / S_ERROR / S_SUCCESS
  m_slot_iterator.set_page(&vpid)

  while !stop:
    scan_code = m_slot_iterator.next_qualified_slot_with_peek()

    if m_xasl->if_pred:
      eval_pred(); skip if V_FALSE/V_UNKNOWN

    if MERGEABLE_LIST / BUILDVALUE_OPT:
      if m_xasl->scan_ptr (NL join inner side):
        scan_reset_scan_block(scan_ptr->curr_spec->s_id)
        while qexec_execute_scan_ptr() == S_SUCCESS:
          result_handler->write(outptr_list or agg)

    if XASL_SNAPSHOT:
      result_handler->write(m_xasl->val_list)

    clear_xasl_dptr_list(m_xasl, uses_clones)  ← reset per-row dptr state
```

### `finalize(thread_ref)`

```
if on_trace:
  compute elapsed_time from m_start_tick
  if MERGEABLE_LIST / BUILDVALUE_OPT:
    m_trace_handler->m_trace_storage_for_sibling_xasl.merge_xasl_tree(m_xasl)
  m_trace_handler->add_trace(fetches, ioreads, fetch_time, read_rows, qualified_rows, elapsed_time)
  perfmon_destroy_parallel_stats(thread_ref)

m_result_handler->write_finalize()
m_input_handler->finalize()
m_slot_iterator.finalize()

if MERGEABLE_LIST / BUILDVALUE_OPT:
  for each xptr in scan_ptr chain:
    join_info->record_join_info(xptr->header.id, xptr)  ← publish scan status

if XASL_SNAPSHOT:
  scan_end_scan / scan_close_scan

free m_vd->dbval_ptr, m_xasl_state
qexec_clear_xasl(m_xasl, true, false)

lock main_thread.m_px_lock_mutex
  if m_uses_xasl_clone:
    xcache_retire_clone(m_xasl_cache_entry, &m_xasl_clone)
    xcache_unfix(m_xasl_cache_entry)
  else:
    free_xasl_unpack_info(m_xasl_unpack_info)
unlock
```

## `clear_xasl_dptr_list`

Called after each qualified row in the loop. Iterates `m_xasl->dptr_list`:
- If `uses_clones` and `XASL_DECACHE_CLONE` is set: `status = XASL_CLEARED`
- If `uses_clones` and not set: `status = XASL_INITIALIZED` (allows reuse)
- If not `uses_clones`: `status = XASL_CLEARED`
- Truncates non-empty list_id; clears single_tuple values.

## Constraints

- **Memory**: task itself is `malloc`-allocated (not `db_private_alloc`) by `manager::start_tasks()` — it must outlive the main thread's stack frame. Inner allocations (`xasl_state`, `vd->dbval_ptr`, `scan_func_ptr`) use `db_private_alloc` on the worker thread.
- **XASL cache mutex**: `m_px_lock_mutex` serialises clone acquire and retire across all workers. Lock duration is minimal — only covers the `xcache_find_xasl_id_for_execute` call.
- **Threading**: all worker threads share the same `m_interrupt`, `m_input_handler`, `m_result_handler`, and `m_join_info` pointers. These must not be freed until all workers have finished.
- **Interrupt**: checked at top of both outer (`input_handler`) and inner (`slot_iterator`) loops.
- **Build mode**: SERVER_MODE + SA_MODE. Uses `db_private_alloc` which requires `private_heap_id != 0`.

## Lifecycle

```
1. manager::start_tasks(): malloc task, placement_new, worker_manager->push_task
2. worker thread picks up task: execute() called
3. initialize(): clone XASL, open scans, init slot_iterator + input_handler + result_handler write side
4. loop(): scan pages, iterate slots, write results
5. finalize(): trace, write_finalize, join_info update, XASL release
6. retire(): worker_manager->pop_task(), delete this
```

## Related

- [[components/parallel-heap-scan|parallel-heap-scan]] — parent hub
- [[components/parallel-heap-scan-slot-iterator|parallel-heap-scan-slot-iterator]] — embedded in task
- [[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]] — VPID source
- [[components/parallel-heap-scan-result-handler|parallel-heap-scan-result-handler]] — result sink
- [[components/parallel-heap-scan-join-info|parallel-heap-scan-join-info]] — NL join coordination
- [[components/xasl|xasl]] — XASL cache, clone, find_by_id, clear
- [[components/mvcc|mvcc]] — heap_next_1page respects MVCC via borrowed tran_index
- [[components/btree|btree]] — level 1+ index scans opened in initialize
- [[Memory Management Conventions]] — mixed `malloc` / `db_private_alloc` pattern
