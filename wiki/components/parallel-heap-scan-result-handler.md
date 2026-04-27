---
status: developing
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_heap_scan/px_heap_scan_result_handler.hpp"
path_impl: "(implementations inline / in px_heap_scan.cpp)"
tags:
  - component
  - cubrid
  - parallel
  - query
  - heap-scan
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-task|parallel-heap-scan-task]]"
  - "[[components/list-file|list-file]]"
  - "[[components/xasl|xasl]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan_result_handler` — Per-Thread Write + Main-Thread Read

`result_handler<RESULT_TYPE>` is the synchronisation point between parallel worker threads (producers) and the main thread (consumer). Each `RESULT_TYPE` specialisation uses different write/read types, different internal state classes, and different TLS types.

## RESULT_TYPE Taxonomy

| Enum value | Write type | Read type | Description |
|------------|-----------|-----------|-------------|
| `MERGEABLE_LIST` | `OUTPTR_LIST` | `QFILE_LIST_ID` | Each worker builds its own list file; main merges |
| `XASL_SNAPSHOT` | `VAL_LIST` | `VAL_LIST` | Row-by-row handoff via `list_id_header` descriptors |
| `BUILDVALUE_OPT` | `AGGREGATE_TYPE` | `AGGREGATE_TYPE` | Per-worker partial aggregate (COUNT/MIN/MAX/SUM/AVG/STDDEV*/VAR*); main thread merges via `qdata_aggregate_accumulator_to_accumulator`. Renamed from `COUNT_DISTINCT` and broadened in [[prs/PR-7049-parallel-buildvalue-heap|PR #7049]] (`65d6915`, 2026-04-27). |

## Class / Function Inventory

### Primary template `result_handler<RESULT_TYPE>` (covers MERGEABLE_LIST and XASL_SNAPSHOT)

| Method | Caller | Description |
|--------|--------|-------------|
| `result_handler(query_id, interrupt*, err_messages*, parallelism, g_agg_domain_resolve_need, orig_xasl)` | manager::open() | Ctor |
| `read_initialize(thread_p)` | manager::next() first call | Sets up read cursor (MERGEABLE_LIST: creates list_id_headers; XASL_SNAPSHOT: initialises read_specs) |
| `read(thread_p, dest)` | manager::next() | Blocks on CV until a worker result is available; returns S_SUCCESS / S_END / S_ERROR |
| `read_finalize(thread_p)` | manager::reset() / end() | Tears down read-side state |
| `write_initialize(thread_p, outptr_list, curr_xasl, vd)` | task::initialize() | Allocates TLS write buffer |
| `write(thread_p, src)` | task::loop() per qualified row | Writes row to TLS list; returns false on interrupt |
| `write_finalize(thread_p)` | task::finalize() | Publishes TLS result to shared collection |

### Full specialisation `result_handler<BUILDVALUE_OPT>` (renamed from `COUNT_DISTINCT` in PR #7049)

| Method | Caller | Description |
|--------|--------|-------------|
| `result_handler(query_id, interrupt*, err_messages*, parallelism, orig_agg_list)` | manager::open() | Ctor |
| `read_initialize(thread_p)` | manager::next() | Sets up aggregate read |
| `read(thread_p, dest*)` | manager::next() | CV-wait; merges worker aggregates into `dest` |
| `read_finalize(thread_p)` | manager teardown | Cleanup |
| `write_initialize(thread_p, outptr_list, agg_list, vd, xasl)` | task::initialize() | Creates per-worker TLS aggregate clone |
| `write(thread_p) -> bool` | task::loop() | Accumulates row into TLS aggregate |
| `write_finalize(thread_p)` | task::finalize() | Publishes accumulated aggregate |

### Supporting classes

| Class | Role |
|-------|------|
| `mergeable_list_variables` | Shared state for MERGEABLE_LIST: `writer_results` vector, `active_results` count, domain-resolve flags, hgby results |
| `xasl_snapshot_variables` | Shared state for XASL_SNAPSHOT: `list_id_headers` vector, `read_specs` vector, atomic index |
| `mergeable_list_tls` | TLS for MERGEABLE_LIST workers: `writer_result_p`, `vd`, `xasl`, `agg_hash_state`, domain-resolve state |
| `xasl_snapshot_tls` | TLS for XASL_SNAPSHOT workers: `list_id_header_p`, `tpl_buf` |
| `list_id_header` | Lock-free header for a worker's list file: atomic first/last VPID, closed flag, valid flag |
| `read_spec` | Read cursor: pointer to `list_id_header`, `read_ended`, `list_scan_id` |
| `VPID64_t` | Union of `uint64_t` and `VPID` — enables 64-bit atomic VPID operations |

## Template Type Aliases

```cpp
// Inside result_handler<result_type>:
using read_dest_type  = conditional_t<result_type == MERGEABLE_LIST, QFILE_LIST_ID, VAL_LIST>;
using write_dest_type = conditional_t<result_type == MERGEABLE_LIST, OUTPTR_LIST, VAL_LIST>;
using variables       = conditional_t<result_type == MERGEABLE_LIST, mergeable_list_variables, xasl_snapshot_variables>;
using tls             = conditional_t<result_type == MERGEABLE_LIST, mergeable_list_tls, xasl_snapshot_tls>;
```

> [!key-insight] `XASL_SNAPSHOT` reuses the MERGEABLE_LIST template with different type aliases
> `XASL_SNAPSHOT` uses `xasl_snapshot_variables` and `xasl_snapshot_tls` instead of the mergeable-list variants. The primary template compiles both because all methods are available on both type aliases; the `if constexpr` branches in `task::loop` select the right write call.

## `list_id_header` — Lock-Free List Head Tracking

```cpp
class list_id_header {
    atomic<VPID64_t> m_first_vpid;   // atomic 64-bit VPID union
    atomic<VPID64_t> m_last_vpid;
    atomic<bool>     m_list_closed;  // true when worker finishes writing
    atomic<bool>     m_valid;        // true when at least one tuple written
    QFILE_LIST_ID*   m_list_id_p;
    vector<atomic<TP_DOMAIN*>*> m_type_list;
    int m_type_cnt;
};
```

> [!key-insight] Main thread can start reading before workers finish writing
> `m_first_vpid` becomes non-null as soon as the worker writes its first page. The main thread's `read()` can start scanning that partial result without waiting for `m_list_closed`. This overlaps I/O between producer writes and consumer reads.

## Execution Path — MERGEABLE_LIST

### Worker write path
```
write_initialize(thread_p, outptr_list, curr_xasl, vd)
  └─ TLS: open new QFILE_LIST_ID (tls.writer_result_p), set vd, xasl

write(thread_p, outptr_list)  [called per qualified row]
  └─ qfile_add_tuple_to_list(tls.writer_result_p, ...)
     update list_id_header.m_first_vpid / m_last_vpid atomically on first write

write_finalize(thread_p)
  ├─ qfile_close_list(tls.writer_result_p)
  └─ lock writer_results_mutex
       push tls.writer_result_p into m_.writer_results
       decrement m_.active_results
       m_result_cv.notify_all()
```

### Main read path
```
read_initialize(thread_p)
  └─ allocate list_id_headers[] and read_specs[]; m_.active_results = parallelism

read(thread_p, dest_list_id) [called per scan_next iteration]
  ├─ get_valid_read_spec() — find next non-exhausted spec
  ├─ if no spec ready: m_result_cv.wait() until active_results decremented
  └─ scan through list_id: scan_next_list_scan into QFILE_LIST_SCAN_ID
       when m_list_closed: close list scan, mark read_spec.read_ended
       return S_SUCCESS per tuple, S_END when all workers done
```

## Execution Path — BUILDVALUE_OPT (post-PR #7049)

> [!update] PR #7049 (`65d6915`, 2026-04-27)
> Was `COUNT_DISTINCT`, supported only `COUNT(*)` and `COUNT(col)`. Now supports the full set: `COUNT_STAR`, `COUNT`, `MIN`, `MAX`, `SUM`, `AVG`, `STDDEV`, `STDDEV_POP`, `STDDEV_SAMP`, `VARIANCE`, `VAR_POP`, `VAR_SAMP` (incl. `DISTINCT` variants except for MIN/MAX where `DISTINCT` is a no-op). See [[prs/PR-7049-parallel-buildvalue-heap]] for the full diff walkthrough.

```
write_initialize(thread_p, outptr_list, agg_list, vd, xasl):
  ├─ Force agg_domains_resolved=0 (re-resolve per worker)
  ├─ For each agg_node:
  │    PT_COUNT_STAR              → curr_cnt = 0
  │    Q_DISTINCT (non-MIN/MAX)   → open per-thread QFILE_LIST_ID for operand domain
  │                                 (NEW: alloc/open failures → move_top_error + interrupt)
  │    everything else            → curr_cnt = 0  (value/value2 lazy on first row)

write(thread_p)  [per qualified row]:
  ├─ fetch_peek_dbval (or TYPE_CONSTANT shortcut) for first operand
  ├─ DB_IS_NULL → continue
  ├─ Q_DISTINCT (non-MIN/MAX) → write each operand to per-thread list (de-dup at merge)
  └─ switch on function:
       PT_COUNT     → curr_cnt++
       PT_MIN/MAX   → cmpval (collation-aware); if better OR first row:
                      pr_clear_value, then coerce or pr_clone, curr_cnt++
       PT_SUM/AVG   → first row: clone (with optional coerce); else: qdata_add_dbval; curr_cnt++
       STDDEV/VAR   → tp_value_coerce + qdata_multiply_dbval (X²);
                      first: setval(value, value2); else: add to both halves; curr_cnt++

write_finalize(thread_p)  [under writer_results_mutex]:
  ├─ NEW interrupt check at top of each iter — if interrupted, free remaining lists+values, break
  ├─ For each (orig, worker) pair:
  │    PT_COUNT_STAR → orig.curr_cnt += worker.curr_cnt
  │    Q_DISTINCT (non-MIN/MAX) → qfile_connect_list (with malloc-failure handling)
  │    PT_COUNT → orig.curr_cnt += worker.curr_cnt
  │    everything else (the new fast path):
  │      ├─ if orig.value_dom NULL but worker has it, copy worker's domain
  │      ├─ db_change_private_heap(thread_p, 0)
  │      ├─ qdata_aggregate_accumulator_to_accumulator(orig, &orig_dom, function, domain, &worker)
  │      └─ restore previous heap

read_initialize(thread_p):
  └─ For each agg: only PT_COUNT_STAR/PT_COUNT get value_dom=tp_Bigint_domain
                   (others keep their resolved domain)

read(thread_p, dest):
  ├─ CV-wait until m_result_completed == m_parallelism
  └─ For each agg:
       PT_COUNT_STAR / Q_DISTINCT → no-op
       PT_COUNT → db_make_bigint(value, curr_cnt)
       everything else → two-heap dance (see below)
```

### Cross-thread DB_VALUE ownership (heap 0 ↔ private heap)

This is the **key engineering pattern** added by PR #7049:

- **Workers write into heap 0** (process-wide allocator) for accumulator `value` / `value2` so the storage survives the worker thread's teardown when its private heap is destroyed.
- **`write_finalize` merges in heap 0**: `db_change_private_heap(thread_p, 0)` → `qdata_aggregate_accumulator_to_accumulator(...)` → restore. Result still lives in heap 0.
- **`read` (main thread) re-clones into the calling thread's private heap**:
  1. `pr_clone_value(acc->value, &tmp)` — clone defaults to current thread's private heap.
  2. `db_change_private_heap(thread_p, 0)` → `pr_clear_value(acc->value)` — free the heap-0 original.
  3. Restore previous heap.
  4. `*acc->value = tmp` — assign clone back.
  5. Symmetric for `acc->value2` (used by STDDEV/VAR).
- **Why the dance is mandatory**: downstream `qexec_end_buildvalueblock_iterations` calls `pr_clear_value` on accumulators expecting them to be in the **calling thread's** private heap. Without re-cloning, that cleanup would either leak (heap 0 not freed) or crash (wrong-heap free).
- `qdata_aggregate_accumulator_to_accumulator` is the standard CUBRID accumulator merge primitive — same one used by hash GROUP BY and serial aggregation. Reused here to avoid duplicating per-aggregate merge logic.

### Aggregate function whitelist (`is_buildvalue_opt_supported_function`)

Defined in `px_heap_scan_checker.cpp`. Returns `true` for: `PT_COUNT_STAR`, `PT_COUNT`, `PT_MIN`, `PT_MAX`, `PT_SUM`, `PT_AVG`, `PT_STDDEV`, `PT_STDDEV_POP`, `PT_STDDEV_SAMP`, `PT_VARIANCE`, `PT_VAR_POP`, `PT_VAR_SAMP`. Anything else (e.g. `PT_GROUP_CONCAT`, `PT_MEDIAN`, `PT_JSON_*`, bit aggregates) falls back to `MERGEABLE_LIST` / `XASL_SNAPSHOT`.

### MIN/MAX-DISTINCT shortcut

`agg_node->option == Q_DISTINCT && agg_node->function != PT_MIN && agg_node->function != PT_MAX` — MIN/MAX with DISTINCT is semantically equivalent to MIN/MAX without it (extrema don't care about duplicates). Excluded from the per-thread DISTINCT-list path to avoid wasted work.

## `mergeable_list_variables` Fields

| Field | Type | Role |
|-------|------|------|
| `writer_results` | `vector<QFILE_LIST_ID*>` | One entry per worker; populated by write_finalize |
| `writer_results_mutex` | `mutex` | Guards append to writer_results |
| `orig_xasl` | `XASL_NODE*` | For domain resolution |
| `active_results` | `int` | Counts workers still writing; CV wakes on decrement |
| `is_list_id_domain_resolved` | `bool` | One-time domain resolve flag |
| `hgby_results` | `vector<QFILE_LIST_ID*>` | GROUP BY hash results from workers |
| `g_hash_eligible` | `bool` | Whether hash GROUP BY was used |

## Constraints

- **Threading**: `m_result_mutex` / `m_result_cv` are the primary producer-consumer synchronisation. `writer_results_mutex` guards the results vector.
- **Memory**: `QFILE_LIST_ID` objects are allocated by `qfile_open_list` in the worker's context using the query's temp file. They are closed and freed by the main thread after merging.
- **Interrupt**: workers call `m_interrupt_p->get_code()` before each write; main calls it in the `read` CV-wait condition. On interrupt, workers call `write_finalize` early to ensure `active_results` reaches 0 and the main thread is unblocked.
- **Domain resolution**: MERGEABLE_LIST has special handling for `g_agg_domain_resolve_need` — the first worker to complete resolves domain types for the aggregation via `qexec_resolve_domains_for_aggregation_for_parallel_heap_scan_g_agg`.

## Lifecycle

```
1. manager::open() constructs result_handler<T>
2. task::initialize() calls write_initialize (per worker)
3. task::loop() calls write() per qualified row
4. task::finalize() calls write_finalize — publishes result, decrements counter
5. manager::next() calls read_initialize (first call), then read() per iteration
6. manager::reset() / end() calls read_finalize
7. manager destructor: result_handler::read_finalize + ~result_handler + db_private_free
```

## Related

- [[components/parallel-heap-scan|parallel-heap-scan]] — parent hub
- [[components/parallel-heap-scan-task|parallel-heap-scan-task]] — calls write / write_initialize / write_finalize
- [[components/list-file|list-file]] — `QFILE_LIST_ID`, `qfile_*` operations
- [[components/xasl|xasl]] — `VAL_LIST`, `OUTPTR_LIST`, `AGGREGATE_TYPE`
- [[Memory Management Conventions]] — `db_private_alloc` ownership rules
