---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_heap_scan/px_heap_scan_slot_iterator.hpp"
path_impl: "src/query/parallel/px_heap_scan/px_heap_scan_slot_iterator.cpp"
tags:
  - component
  - cubrid
  - parallel
  - query
  - heap-scan
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-task|parallel-heap-scan-task]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/mvcc|mvcc]]"
  - "[[components/storage|storage]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan_slot_iterator` — Slot Iteration Within a Heap Page

`slot_iterator` iterates all qualifying slots on a single heap page. It is embedded (by value) inside each `task<T>` and operates on one page at a time, as directed by `input_handler_ftabs`.

## Purpose

After `input_handler_ftabs` delivers a fixed heap page (VPID), `slot_iterator` walks all record slots on that page calling `heap_next_1page`, applies the data filter predicate (`eval_data_filter`), reads "rest" attributes, and fetches the value list — returning `S_SUCCESS` per qualified row.

## Class / Function Inventory

| Method | Description |
|--------|-------------|
| `slot_iterator()` | Default ctor; nulls all pointers and sets `m_is_peeking = false` |
| `~slot_iterator()` | Empty dtor |
| `initialize(thread_p, scan_id, vd)` | Extracts predicate filter, scan cache, OID, class OID, rest regu list, val list from `HEAP_SCAN_ID`; sets `m_is_peeking = (scan_id->fixed != 0)` |
| `finalize(thread_p)` | No-op (resource cleanup happens at `SCAN_ID` / scan_cache level) |
| `set_page(thread_p, vpid*)` | Sets `m_vpid = *vpid`; resets `m_cur_oid` to NULL_PAGEID; asserts `scan_cache->page_watcher.pgptr != nullptr` |
| `next_qualified_slot_with_peek(thread_p)` | Main slot-advance method (see below); returns S_SUCCESS / S_END / S_ERROR |

### Key Fields

| Field | Type | Role |
|-------|------|------|
| `m_data_filter` | `FILTER_INFO` | Predicate filter built from `hsidp->scan_pred` and `hsidp->pred_attrs` |
| `m_is_peeking` | `bool` | Whether to use PEEK mode (true if scan is fixed) |
| `m_cur_oid` | `OID` | Current record OID (advanced by `heap_next_1page`) |
| `m_class_oid` | `OID` | Class OID (for `eval_data_filter`) |
| `m_next_oid` | `OID` | Not used in current implementation |
| `m_recdes` | `RECDES` | Record descriptor reset per slot |
| `m_ref_lsa` | `LOG_LSA` | LSA copied after page fix for change detection |
| `m_rest_regu_list` | `regu_variable_list_node*` | Attributes to fetch after predicate passes |
| `m_rest_attr_cache` | `HEAP_CACHE_ATTRINFO*` | Attribute info cache for rest attributes |
| `m_val_list` | `VAL_LIST*` | Destination value list |
| `m_scan_cache` | `HEAP_SCANCACHE*` | Points to worker's scan cache (contains `page_watcher`) |
| `m_vpid` | `VPID` | Current page being iterated |
| `m_hfid` | `HFID` | Heap file ID |
| `m_vd` | `val_descr*` | Value descriptor for `fetch_val_list` |
| `m_scan_stats` | `SCAN_STATS*` | `read_rows` / `qualified_rows` counters |
| `m_on_trace` | `bool` | Whether to update scan stats |

## Execution Path — `next_qualified_slot_with_peek`

```
while (1):
  COPY_OID(&retry_oid, &m_cur_oid)

restart_scan_oid:
  m_recdes = RECDES_INITIALIZER
  slot_code = heap_next_1page(thread_p, &m_hfid, &m_vpid, &m_class_oid,
                               &m_cur_oid, &m_recdes, m_scan_cache, m_is_peeking)
  if slot_code == S_END → return S_END
  if slot_code != S_SUCCESS → return S_ERROR

  if on_trace: m_scan_stats->read_rows++

  copy ref_lsa from page_watcher.pgptr

  ev_res = eval_data_filter(thread_p, &m_cur_oid, &m_recdes, m_scan_cache, &m_data_filter)
  if on_trace: m_scan_stats->qualified_rows++

  if ev_res == V_ERROR → return S_ERROR
  if is_peeking == PEEK and page changed since ref_lsa:
    is_peeking = COPY; COPY_OID(&m_cur_oid, &retry_oid); goto restart_scan_oid

  if ev_res != V_TRUE → continue   (next slot)

  if m_rest_regu_list:
    heap_attrinfo_read_dbvalues(m_cur_oid, m_recdes, m_rest_attr_cache)
    if page changed: is_peeking = COPY; goto restart_scan_oid
    if m_val_list:
      fetch_val_list(m_rest_regu_list, m_vd, &m_class_oid, &m_cur_oid, NULL, PEEK)
      if page changed: is_peeking = COPY; goto restart_scan_oid

  return S_SUCCESS
```

> [!key-insight] PEEK-to-COPY demotion on concurrent page modification
> When `m_is_peeking = true` (fixed scan), the iterator checks `PGBUF_IS_PAGE_CHANGED(page_watcher.pgptr, &m_ref_lsa)` after every step that touches the page. If a concurrent writer has modified the page between the predicate evaluation and the attribute fetch, the slot is retried with `is_peeking = COPY`. This ensures attribute values and predicate results are consistent from a single physical read.

> [!key-insight] `heap_next_1page` vs. `heap_next`
> The parallel slot iterator calls `heap_next_1page` (not `heap_next`), which is bounded to a single page. The inter-page navigation is handled externally by `input_handler_ftabs::get_next_vpid_with_fix`. This clean separation allows each worker to own a VPID and iterate it independently.

## MVCC Integration

`heap_next_1page` applies MVCC visibility internally: only records visible to the current MVCC snapshot (as determined by the worker's borrowed `tran_index`) are returned. The slot iterator does not explicitly call `mvcc_satisfies_snapshot` — it delegates to the standard heap scan machinery.

> [!warning] `m_scan_cache->page_watcher.pgptr` must not be null at `set_page`
> The assertion `assert(m_scan_cache->page_watcher.pgptr != nullptr)` in `set_page` fires if `input_handler_ftabs::get_next_vpid_with_fix` returned a VPID for a page that was not successfully fixed. This would be a bug in the input handler — it should only return S_SUCCESS with a fixed page.

## Constraints

- **Single-page scope**: `slot_iterator` only iterates one page per `set_page` call. The task loop drives it across pages.
- **No independent locking**: the page latch is held by `m_scan_cache->page_watcher` (managed by `input_handler_ftabs`). The slot iterator reads from the latched page without additional locking.
- **Embedded object**: `slot_iterator` is value-embedded in `task<T>` — no heap allocation. Initialize/finalize are idempotent and lightweight.
- **Build mode**: SERVER_MODE + SA_MODE.

## Lifecycle

```
1. task::initialize() calls slot_iterator::initialize(scan_id, vd) — extracts state from SCAN_ID
2. task::loop():
   a. input_handler::get_next_vpid_with_fix → fixes page in scan_cache.page_watcher
   b. slot_iterator::set_page(&vpid) — records current VPID
   c. [inner loop] slot_iterator::next_qualified_slot_with_peek() per slot
3. task::finalize() calls slot_iterator::finalize() — no-op
```

## Related

- [[components/parallel-heap-scan-task|parallel-heap-scan-task]] — embeds slot_iterator; drives set_page / next_qualified_slot_with_peek
- [[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]] — delivers fixed page VPID
- [[components/heap-file|heap-file]] — `heap_next_1page`, `heap_attrinfo_read_dbvalues`, `HEAP_SCANCACHE`
- [[components/mvcc|mvcc]] — MVCC filtering inside `heap_next_1page`
- [[components/storage|storage]] — `PGBUF_IS_PAGE_CHANGED`, `LOG_LSA`, latch model
