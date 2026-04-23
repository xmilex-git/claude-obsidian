---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_heap_scan/"
status: developing
purpose: "Parallel full heap scan with per-worker XASL clones, page-range distribution, and three result collection modes"
key_files:
  - "px_heap_scan.hpp (manager<RESULT_TYPE> template + C extern API)"
  - "px_heap_scan.cpp (C-extern dispatch: scan_open/start/next/reset/end/close_parallel_heap_scan)"
  - "px_heap_scan_result_type.hpp (RESULT_TYPE enum)"
  - "px_heap_scan_result_handler.hpp (result_handler<T> template with TLS write + CV-wait read)"
  - "px_heap_scan_input_handler_ftabs.hpp (page-set splitting and TLS VPID iteration)"
  - "px_heap_scan_task.hpp (task<T>: XASL clone per worker, slot iteration loop)"
  - "px_heap_scan_join_info.hpp (join_info: cross-XASL scan state for joins)"
  - "px_heap_scan_trace_handler.hpp (per-worker stats accumulation)"
  - "px_heap_scan_ftab_set.hpp (ftab_set: page sector table split helper)"
public_api:
  - "scan_open_parallel_heap_scan(...) [C extern]"
  - "scan_start_parallel_heap_scan(...) [C extern]"
  - "scan_next_parallel_heap_scan(...) -> SCAN_CODE [C extern]"
  - "scan_reset_scan_block_parallel_heap_scan(...) [C extern]"
  - "scan_end_parallel_heap_scan(...) [C extern]"
  - "scan_close_parallel_heap_scan(...) [C extern]"
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
  - "[[components/storage|storage]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan/` — Parallel Heap Scan

Parallelises full-table scans by distributing heap page sectors across worker threads, each running an independent XASL clone. Results are collected in one of three modes determined at open time.

## Three Result Modes

```cpp
enum class RESULT_TYPE {
    MERGEABLE_LIST = 0x1,   // (fast)  each worker writes its own list file, main merges
    XASL_SNAPSHOT  = 0x2,   // (slow)  row-by-row handoff via shared headers
    COUNT_DISTINCT = 0x3,   // (fast)  for UPDATE STATISTICS (aggregate-only)
};
```

> [!key-insight] Template-based specialisation, not runtime polymorphism
> `manager<RESULT_TYPE>`, `task<RESULT_TYPE>`, and `result_handler<RESULT_TYPE>` are all C++ templates. The C extern functions (`scan_next_parallel_heap_scan`) dispatch via a switch on `scan_id->s.phsid.result_type` to the right template instantiation. This avoids virtual dispatch overhead in the hot path.

## Lifecycle (extern C API maps to manager methods)

```
scan_open_parallel_heap_scan  → manager::open()      (reserve workers, init handlers)
scan_start_parallel_heap_scan → manager::start_tasks() (split ftab_set, dispatch tasks)
scan_next_parallel_heap_scan  → manager::next()       (read from result_handler)
scan_reset_scan_block_*       → manager::reset()
scan_end_parallel_heap_scan   → manager::end()        (merge stats)
scan_close_parallel_heap_scan → manager::close()      (release workers, free memory)
```

## Input Distribution — `input_handler_ftabs`

The heap file's file-table (ftab) is fetched on the main thread (`init_on_main`) and split into `parallelism` equal `ftab_set` slices. Each worker atomically claims its slice via `m_splited_ftab_set_idx` (atomic int, fetch-add pattern).

Workers iterate heap pages via `get_next_vpid_with_fix`, using thread-local storage:

| TLS field | Role |
|-----------|------|
| `m_tl_vpid` | Current VPID |
| `m_tl_scan_cache` | Per-thread `HEAP_SCANCACHE` |
| `m_tl_old_page_watcher` | `PGBUF_WATCHER` for page pinning |
| `m_tl_ftab_set` | Pointer to this worker's `ftab_set` slice |
| `m_tl_pgoffset` | Offset within current sector |

## Result Collection — `result_handler<T>`

Writers (workers) and reader (main) use a split interface:

| Method | Called by | Mode |
|--------|-----------|------|
| `write_initialize(thread_p, ...)` | Worker startup | Allocates TLS write buffer |
| `write(thread_p, src)` | Worker hot loop | Writes tuples to TLS list |
| `write_finalize(thread_p)` | Worker teardown | Publishes list to shared vector |
| `read_initialize(thread_p)` | Main, after start_tasks | Sets up read cursor |
| `read(thread_p, dest)` | Main, per scan_next call | CV-waits on available worker result |
| `read_finalize(thread_p)` | Main, at end | Cleans up |

### MERGEABLE_LIST internals

Each worker writes into its own `QFILE_LIST_ID` (in `mergeable_list_tls::writer_result_p`). On `write_finalize`, the list ID is pushed into `mergeable_list_variables::writer_results` under `writer_results_mutex`. The main thread reads them in order via `list_id_header` descriptors (atomic VPID pointers marking first/last page of each sub-list).

```
list_id_header
  ├── m_first_vpid : atomic<VPID64_t>
  ├── m_last_vpid  : atomic<VPID64_t>
  ├── m_list_closed: atomic<bool>
  └── m_valid      : atomic<bool>
```

> [!key-insight] Lock-free list head tracking
> Workers update `m_first_vpid` and `m_last_vpid` atomically so the main thread can start reading a worker's partial result before it finishes writing. `m_list_closed` signals the reader that the last page is stable.

## Per-Worker XASL Clone — `task<T>`

Each worker:
1. Clones the XASL tree from the XASL cache (`clone_xasl`).
2. Opens its own `SCAN_ID` over the heap.
3. Runs `loop()`: calls `get_next_vpid_with_fix` (input), evaluates predicates, calls `handle_result` (output).
4. On interrupt or end: calls `finalize()`, signals `worker_manager::pop_task()`.

XASL cloning ensures worker threads have independent predicate evaluation state, value descriptors (`val_descr`), and scan function pointers.

## Join Info — `join_info`

When parallel heap scan is used within a join (e.g., NL join outer), `join_info` captures the scan state (OID, HFID, BTID, access method) for each XASL node in the plan. Workers call `apply_join_info` to synchronise scan status across XASL siblings (protected by `m_mutex`).

## Tracing

Each worker has a `trace_handler` accumulating `child_stats` (fetches, ioreads, read_rows, qualified_rows, elapsed_time). At `merge_stats()`, the manager calls `trace_handler::merge_stats()` into the `SCAN_STATS` of the original scan. Trace output supports both text and JSON formats (`dump_stats_text`, `dump_stats_json`).

## Related

- [[components/parallel-query|parallel-query]] — degree selection, interrupt protocol
- [[components/parallel-worker-manager|parallel-worker-manager]] — worker reservation and dispatch
- [[components/storage|storage]] — heap files and page buffer accessed by workers
- Source: [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]
