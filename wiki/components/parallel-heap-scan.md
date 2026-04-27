---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_heap_scan/"
status: active
purpose: "Parallel full heap scan with per-worker XASL clones, page-range distribution via file table, and three result collection modes"
key_files:
  - "px_heap_scan.hpp — manager<RESULT_TYPE> template + C extern API"
  - "px_heap_scan.cpp — C-extern dispatch + manager method implementations"
  - "px_heap_scan_result_type.hpp — RESULT_TYPE enum"
  - "px_heap_scan_checker.hpp/.cpp — XASL feasibility check"
  - "px_heap_scan_ftab_set.hpp — sector table split helper"
  - "px_heap_scan_input_handler_ftabs.hpp/.cpp — page-set distribution to workers"
  - "px_heap_scan_result_handler.hpp — result_handler<T> with TLS write + CV-wait read"
  - "px_heap_scan_task.hpp/.cpp — task<T>: per-worker XASL clone + scan loop"
  - "px_heap_scan_slot_iterator.hpp/.cpp — per-page slot iteration with MVCC peek"
  - "px_heap_scan_join_info.hpp/.cpp — scan state for NL joins"
  - "px_heap_scan_trace_handler.hpp/.cpp — per-worker stats accumulation"
public_api:
  - "scan_open_parallel_heap_scan(...) [C extern]"
  - "scan_start_parallel_heap_scan(...) [C extern]"
  - "scan_next_parallel_heap_scan(...) -> SCAN_CODE [C extern]"
  - "scan_reset_scan_block_parallel_heap_scan(...) [C extern]"
  - "scan_end_parallel_heap_scan(...) [C extern]"
  - "scan_close_parallel_heap_scan(...) [C extern]"
  - "scan_check_parallel_heap_scan_possible(xasl) [C extern]"
tags:
  - component
  - cubrid
  - parallel
  - query
  - heap-scan
related:
  - "[[modules/src|src]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-worker-manager|parallel-worker-manager]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/storage|storage]]"
  - "[[components/xasl|xasl]]"
  - "[[components/mvcc|mvcc]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan/` — Parallel Heap Scan (Hub)

Parallelises full-table heap scans by distributing heap file sectors across worker threads, each running an independent XASL clone. Result collection is controlled by a `RESULT_TYPE` template parameter chosen at `scan_open` time.

## Sub-pages

| Page | Content |
|------|---------|
| [[components/parallel-heap-scan-input-handler]] | `ftab_set` + `input_handler_ftabs`: sector splitting and TLS VPID iteration |
| [[components/parallel-heap-scan-result-handler]] | `result_handler<T>` + aux types: TLS write, CV-wait read, three specialisations |
| [[components/parallel-heap-scan-task]] | `task<T>`: XASL clone, `slot_iterator` loop, write to result handler |
| [[components/parallel-heap-scan-slot-iterator]] | `slot_iterator`: single-page heap slot iteration with MVCC peek |
| [[components/parallel-heap-scan-join-info]] | `join_info`: cross-worker scan-state synchronisation for NL joins |
| [[components/parallel-heap-scan-support]] | `checker` (XASL feasibility) + `trace_handler` (stats accumulation) |

## Three Result Modes

```cpp
enum class RESULT_TYPE {
    NONE           = 0x0,
    MERGEABLE_LIST = 0x1,  // (fast)  per-thread list file, main merges — set-independent SELECT
    XASL_SNAPSHOT  = 0x2,  // (slow)  row-by-row handoff via shared list_id_headers — set-dependent
    BUILDVALUE_OPT = 0x3,  // (fast)  BUILDVALUE_PROC fast path: per-worker partial aggregate (PR #7049)
};
```

> [!key-insight] Template-based specialisation, not runtime polymorphism
> `manager<RESULT_TYPE>`, `task<RESULT_TYPE>`, and `result_handler<RESULT_TYPE>` are C++ templates with explicit instantiations for all three non-NONE variants. The C extern functions dispatch via a `switch` on `scan_id->s.phsid.result_type`. This eliminates virtual dispatch overhead in the hot scan path.

## Result Mode Selection (in `scan_open_parallel_heap_scan`)

> [!update] PR #7049 (`65d6915`, 2026-04-27)
> `COUNT_DISTINCT` enum value renamed to `BUILDVALUE_OPT` (same bit 0x3); `ACCESS_SPEC_FLAG_COUNT_DISTINCT` renamed to `ACCESS_SPEC_FLAG_BUILDVALUE_OPT` (same bit `0x1 << 4`). The mode now gates the full set of order-independent aggregates (COUNT, MIN, MAX, SUM, AVG, STDDEV*, VAR*) instead of just COUNT(*) / COUNT(col). See [[prs/PR-7049-parallel-buildvalue-heap]] for details.

```
ACCESS_SPEC_FLAG_MERGEABLE_LIST set?   → MERGEABLE_LIST
ACCESS_SPEC_FLAG_BUILDVALUE_OPT set?   → BUILDVALUE_OPT
else                                   → XASL_SNAPSHOT

Exceptions that force XASL_SNAPSHOT:
  xasl->topn_items != nullptr          (TOP-N queries)
  XASL_IS_FLAGED(xasl, XASL_TO_BE_CACHED)
```

## Top-Level Lifecycle (extern C API → manager methods)

```
scan_open_parallel_heap_scan
  ├─ feasibility check (sys class, MVCC disabled, no private_heap_id → fallback)
  ├─ file_get_num_user_pages → compute_parallel_degree
  ├─ worker_manager::try_reserve_workers(num_parallel_threads)
  ├─ select RESULT_TYPE, db_private_alloc manager, placement_new
  └─ manager::open()
       ├─ qmgr_get_query_entry (determine xasl_clone mode)
       ├─ deep-copy VAL_DESCR (pr_clone_value per dbval)
       ├─ placement_new input_handler_ftabs
       ├─ input_handler::init_on_main (collect + split ftab sectors)
       └─ placement_new result_handler<T>

scan_start_parallel_heap_scan
  └─ scan_id->position = S_ON   (scan_manager integration)

scan_next_parallel_heap_scan → manager::next()
  ├─ first call: start_tasks() — dispatch task<T> workers
  ├─ read_initialize (first call)
  ├─ result_handler::read(thread_p, dest) — CV-wait on worker output
  ├─ join_info::apply_join_info (MERGEABLE_LIST / BUILDVALUE_OPT)
  └─ handle interrupt codes

scan_reset_scan_block_parallel_heap_scan → manager::reset()
  ├─ signal JOB_ENDED interrupt
  ├─ worker_manager::wait_workers()
  ├─ teardown input_handler, result_handler, val_descr
  └─ re-open (manager::open())

scan_end_parallel_heap_scan → manager::end()
  ├─ signal JOB_ENDED
  ├─ worker_manager::release_workers()
  └─ merge_stats() into scan_stats

scan_close_parallel_heap_scan → manager::close()
  └─ ~manager() + db_private_free
```

## Manager Template Fields

| Field | Type | Role |
|-------|------|------|
| `m_thread_p` | `THREAD_ENTRY*` | Main thread |
| `m_scan_id` | `SCAN_ID*` | The heap scan ID |
| `m_xasl` | `xasl_node*` | XASL node being scanned |
| `m_parallelism` | `int` | Number of workers |
| `m_hfid` / `m_cls_oid` | `HFID` / `OID` | Table identity |
| `m_vd` / `m_orig_vd` | `val_descr*` | Deep-copied / original value descriptor |
| `m_input_handler` | `input_handler_ftabs*` | Page sector distributor |
| `m_result_handler` | `result_handler<result_type>*` | Result collector |
| `m_interrupt` | `parallel_query::interrupt` | Cross-worker interrupt signalling |
| `m_atomic_instnum` | `atomic_instnum` | INSTNUM row counter |
| `m_err_messages` | `err_messages_with_lock` | Worker error propagation |
| `m_join_info` | `join_info` | NL join scan-state synchronisation |
| `m_uses_xasl_clone` | `bool` | True if XASL cache has valid SHA-1 hash |
| `m_trace_handler` | `trace_handler` | Per-worker stat accumulation |

## XASL Clone Mode

```
manager::open():
  if query_entry->xasl_id.sha1.h[] == 0:
    m_uses_xasl_clone = false
    each task: stx_map_stream_to_xasl from main_thread.xasl_unpack_info_ptr
  else:
    m_uses_xasl_clone = true
    each task: xcache_find_xasl_id_for_execute + xcache_retire_clone on finalize
```

> [!key-insight] Two XASL clone paths: cache clone vs. stream re-unpack
> When the query is cacheable (non-zero SHA-1), workers clone from the XASL cache — fast. When not cacheable (prepared statements run inline), workers re-unpack the packed stream under a mutex. Both paths call `xasl_find_by_id(clone, m_xasl_id)` to locate the right node in the cloned tree.

## Fallback to Single-Thread

`scan_open_parallel_heap_scan` returns `NO_ERROR` without setting `scan_id->type = S_PARALLEL_HEAP_SCAN` in these cases:
- System class (`oid_is_system_class`)
- MVCC-disabled class (`mvcc_is_mvcc_disabled_class`)
- Lock-needed scan (`mvcc_select_lock_needed`)
- Worker allocation fails (`worker_manager::try_reserve_workers` returns nullptr)
- `num_parallel_threads < 2`
- Any `open()` error except `ER_INTERRUPTED` (which propagates)

## Constraints

- **Build mode**: SERVER_MODE + SA_MODE; `private_heap_id == 0` check prevents use in child threads that lack a private heap.
- **Memory**: `manager` is `db_private_alloc`-ed; `input_handler` and `result_handler` likewise; tasks are `malloc`-ed (not private heap) because they survive beyond the main thread's stack frame.
- **Threading**: worker threads borrow `conn_entry` and `tran_index` from the main thread. MVCC snapshot is identical across all workers.
- **Interrupt**: `m_interrupt` is a shared atomic interrupt code. Workers check it per VPID. Main thread checks it in `next()` and `reset()`.
- **Partitioned classes**: `DB_PARTITIONED_CLASS` is excluded; only `DB_PARTITION_CLASS` partitions are eligible.

## Related

- [[components/parallel-heap-scan-input-handler]] — ftab sector splitting and VPID iteration
- [[components/parallel-heap-scan-result-handler]] — result collection, three specialisations
- [[components/parallel-heap-scan-task]] — per-worker XASL clone and scan loop
- [[components/parallel-heap-scan-slot-iterator]] — per-page slot walk with peek/retry
- [[components/parallel-heap-scan-join-info]] — NL join state synchronisation
- [[components/parallel-heap-scan-support]] — feasibility checker + trace handler
- [[components/parallel-query|parallel-query]] — degree selection, interrupt protocol
- [[components/parallel-worker-manager|parallel-worker-manager]] — worker reservation
- [[components/heap-file|heap-file]] — underlying heap storage
- [[components/xasl|xasl]] — XASL cloning, cache
- [[components/mvcc|mvcc]] — MVCC snapshot shared via tran_index borrowing
