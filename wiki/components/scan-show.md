---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/show_scan.c + show_scan.h"
status: active
purpose: "Virtual scan driver for all SHOW STATISTICS / SHOW TABLES / SHOW THREADS etc. statements — dispatches to per-subsystem start/next/end function triples via a static jump table indexed by SHOWSTMT_TYPE"
key_files:
  - "src/query/show_scan.c — showstmt_scan_init(), showstmt_next_scan(), showstmt_start_scan(), showstmt_end_scan(), SHOWSTMT_ARRAY_CONTEXT helpers, thread_scan_mapfunc"
  - "src/query/show_scan.h — SHOWSTMT_ARRAY_CONTEXT, thread_start_scan(), showstmt_alloc_*"
  - "src/parser/show_meta.h — SHOWSTMT_TYPE enum and SHOWSTMT_METADATA registry (see [[components/show-meta]])"
public_api:
  - "showstmt_scan_init() — one-time init of show_Requests[] dispatch table"
  - "showstmt_start_scan(thread_p, s_id) → int"
  - "showstmt_next_scan(thread_p, s_id) → SCAN_CODE"
  - "showstmt_end_scan(thread_p, s_id) → int"
  - "showstmt_alloc_array_context(thread_p, num_capacity, num_cols) → SHOWSTMT_ARRAY_CONTEXT*"
  - "showstmt_free_array_context(thread_p, ctx)"
  - "showstmt_alloc_tuple_in_context(thread_p, ctx) → DB_VALUE*"
  - "thread_start_scan(thread_p, type, arg_values, arg_cnt, ctx) → int"
tags:
  - component
  - cubrid
  - query
  - scan
  - show
  - diagnostics
related:
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/show-meta|show-meta]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/btree|btree]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `show_scan.c` — SHOW Virtual Scan

`show_scan.c` is the execution engine for all CUBRID `SHOW ...` diagnostic statements. It implements the `S_SHOWSTMT_SCAN` scan type in [[components/scan-manager|scan-manager]] using a static dispatch table (`show_Requests[]`) that maps each `SHOWSTMT_TYPE` enum value to a subsystem-specific start/next/end function triple.

## Purpose

CUBRID exposes internal engine state via SQL-level `SHOW` statements (e.g. `SHOW VOLUME HEADER`, `SHOW INDEX HEADER`, `SHOW THREADS`). These are not real tables; they are virtual scans that call into the relevant subsystem to gather and return rows. `show_scan.c` provides the uniform scan interface that [[components/scan-manager|scan-manager]] expects, while delegating content production to the owning subsystem.

## Public Entry Points

| Function | Phase | Description |
|---|---|---|
| `showstmt_scan_init()` | bootstrap | Fills `show_Requests[SHOWSTMT_END]` once; idempotent (`show_scan_Inited` guard) |
| `showstmt_start_scan(thread_p, s_id)` | start | Calls `show_Requests[show_type].start_func` to materialize rows into `ctx` |
| `showstmt_next_scan(thread_p, s_id)` | next | Calls `show_Requests[show_type].next_func(cursor, out_values, out_cnt, ctx)`, advances `cursor++` |
| `showstmt_end_scan(thread_p, s_id)` | close | Calls `show_Requests[show_type].end_func` to free `ctx` |
| `showstmt_alloc_array_context(thread_p, cap, cols)` | helper | Allocate a `SHOWSTMT_ARRAY_CONTEXT` for subsystems that buffer all rows at start |
| `showstmt_alloc_tuple_in_context(thread_p, ctx)` | helper | Append one `DB_VALUE[num_cols]` row to the context; reallocs at 1.5x growth |
| `showstmt_free_array_context(thread_p, ctx)` | helper | `pr_clear_value` each cell, then `db_private_free` everything |
| `thread_start_scan(thread_p, type, arg_values, arg_cnt, ctx)` | start | Fills `SHOWSTMT_ARRAY_CONTEXT` with one row per live thread entry (SERVER_MODE); uses `cubthread::manager::map_entries` |

## Dispatch Table

`showstmt_scan_init()` populates `show_Requests[]`, a static array of `SHOW_REQUEST` structs indexed by `SHOWSTMT_TYPE`:

```c
typedef struct show_request {
  SHOWSTMT_TYPE    show_type;
  START_SCAN_FUNC  start_func;  // int (*)(thread_p, show_type, arg_values, arg_cnt, ctx**)
  NEXT_SCAN_FUNC   next_func;   // SCAN_CODE (*)(thread_p, cursor, out_values, out_cnt, ctx)
  END_SCAN_FUNC    end_func;    // int (*)(thread_p, ctx**)
} SHOW_REQUEST;
```

Registered entries as of the current source:

| `SHOWSTMT_TYPE` | `start_func` | `next_func` | Owning module |
|---|---|---|---|
| `SHOWSTMT_VOLUME_HEADER` | `disk_volume_header_start_scan` | `disk_volume_header_next_scan` | `src/storage/disk_manager.c` |
| `SHOWSTMT_ACCESS_STATUS` | `css_user_access_status_start_scan` | `showstmt_array_next_scan` | `src/connection/` |
| `SHOWSTMT_ACTIVE_LOG_HEADER` | `log_active_log_header_start_scan` | `log_active_log_header_next_scan` | `src/transaction/log_manager.c` |
| `SHOWSTMT_ARCHIVE_LOG_HEADER` | `log_archive_log_header_start_scan` | `log_archive_log_header_next_scan` | `src/transaction/log_manager.c` |
| `SHOWSTMT_SLOTTED_PAGE_HEADER` | `spage_header_start_scan` | `spage_header_next_scan` | `src/storage/slotted_page.c` |
| `SHOWSTMT_SLOTTED_PAGE_SLOTS` | `spage_slots_start_scan` | `spage_slots_next_scan` | `src/storage/slotted_page.c` |
| `SHOWSTMT_HEAP_HEADER` / `ALL` | `heap_header_capacity_start_scan` | `heap_header_next_scan` | `src/storage/heap_file.c` |
| `SHOWSTMT_HEAP_CAPACITY` / `ALL` | `heap_header_capacity_start_scan` | `heap_capacity_next_scan` | `src/storage/heap_file.c` |
| `SHOWSTMT_INDEX_HEADER` / `CAPACITY` / `ALL_*` | `btree_index_start_scan` | `btree_index_next_scan` | `src/storage/btree.c` |
| `SHOWSTMT_GLOBAL_CRITICAL_SECTIONS` | `csect_start_scan` | `showstmt_array_next_scan` | `src/base/critical_section.c` |
| `SHOWSTMT_JOB_QUEUES` | `css_job_queues_start_scan` | `showstmt_array_next_scan` | `src/connection/` |
| `SHOWSTMT_TIMEZONES` / `FULL` | `tz_timezones_start_scan` / `tz_full_timezones_start_scan` | `showstmt_array_next_scan` | `src/base/tz_support.c` |
| `SHOWSTMT_TRAN_TABLES` | `logtb_descriptors_start_scan` | `showstmt_array_next_scan` | `src/transaction/log_manager.c` |
| `SHOWSTMT_THREADS` | `thread_start_scan` | `showstmt_array_next_scan` | `show_scan.c` itself |
| `SHOWSTMT_PAGE_BUFFER_STATUS` | `pgbuf_start_scan` | `showstmt_array_next_scan` | `src/storage/page_buffer.c` |

## Two Patterns for Subsystem Implementation

Subsystems can implement SHOW scans in two ways:

**Array pattern** (most common):
- `start_func` gathers all rows into a `SHOWSTMT_ARRAY_CONTEXT` allocated via `showstmt_alloc_array_context`.
- `next_func` is always the shared `showstmt_array_next_scan`: does a cursor-indexed lookup into `ctx->tuples[]`, clones each `DB_VALUE`, and returns `S_SUCCESS` or `S_END`.
- `end_func` is always `showstmt_array_end_scan`: calls `showstmt_free_array_context`.

**Streaming pattern** (disk-scan subsystems):
- `start_func` opens the underlying data structure (e.g. `disk_volume_header_start_scan` opens the volume file header page).
- `next_func` reads one row per call (e.g. advances through pages or btree nodes).
- `end_func` closes the underlying scan.

> [!key-insight] Array vs streaming tradeoff
> The array pattern materialises all rows into `db_private_alloc` memory at `start_scan` time — simple to implement but uses O(N) memory for large result sets. Streaming subsystems (disk, heap, btree) avoid this by holding a persistent cursor between `next_scan` calls. New SHOW statement implementations should prefer streaming if the result set can be large.

## SHOWSTMT_ARRAY_CONTEXT

```c
struct showstmt_array_context {
  DB_VALUE **tuples;   // array of DB_VALUE* rows
  int num_cols;        // columns per row
  int num_used;        // rows filled so far
  int num_total;       // allocated capacity
};
```

Growth policy: `db_private_realloc` at `num_new_total = (int)(num_total * 1.5 + 1)`. The initial capacity is passed by the subsystem's `start_func` to `showstmt_alloc_array_context`.

## SHOWSTMT_THREADS Implementation

`thread_start_scan` is SERVER_MODE only. It iterates all `THREAD_ENTRY` objects via `cubthread::manager::map_entries` using the `thread_scan_mapfunc` mapper. Each non-dead entry emits 26 columns (`THREAD_SCAN_COLUMN_COUNT`), including index, thread ID, tran_index, type, status, resume status, current query, lock wait info, and DB_DATETIME timestamps.

> [!warning] SHOW THREADS is SERVER_MODE only
> `thread_start_scan` is wrapped in `#if defined(SERVER_MODE)`. In SA_MODE there is no thread manager dispatcher; the function is still declared in `show_scan.h` but the implementation body is compiled out. Calling it in SA_MODE returns `NO_ERROR` with zero rows.

## DBA Gating

Some SHOW statements are restricted to DBA users. This check is enforced upstream in [[components/show-meta|show-meta]] (`SHOWSTMT_METADATA.only_for_dba`) during the semantic check phase before XASL generation. By the time `show_scan.c` is called, the authorisation has already been verified.

## Execution Path

```
scan_manager (S_SHOWSTMT_SCAN)
  ├── scan_start_scan()
  │     └── showstmt_start_scan(thread_p, s_id)
  │           └── show_Requests[show_type].start_func(thread_p, show_type, arg_values, arg_cnt, &ctx)
  │
  ├── [loop] scan_next_scan()
  │     └── showstmt_next_scan(thread_p, s_id)
  │           ├── pr_clear_value each out_value (free prior row)
  │           └── show_Requests[show_type].next_func(thread_p, cursor++, out_values, out_cnt, ctx)
  │
  └── scan_close_scan() / scan_end_scan()
        └── showstmt_end_scan(thread_p, s_id)
              └── show_Requests[show_type].end_func(thread_p, &ctx)
```

## Constraints

- **No parallelism**: SHOW scans are always single-threaded; `SHOWSTMT_SCAN_ID` has no parallel state.
- **No filter pushdown**: WHERE predicates on SHOW results are applied by the executor after `showstmt_next_scan` returns a row, not inside the scan itself.
- **Memory**: All rows in the array pattern live in the calling thread's `db_private_alloc` heap. The context is freed in `end_scan`, not during `next_scan`.
- **scan_op_type**: Forward-only sequential scan; cursor is an `int` that monotonically increments.
- **Registering new SHOW flavors**: Add a `SHOWSTMT_TYPE` entry to `showstmt_scan_init()` before the `show_scan_Inited = true` line. The `SHOWSTMT_METADATA` registry in [[components/show-meta|show-meta]] must also be updated.

## Lifecycle

```
open  : scan_open_showstmt_scan() in scan_manager.c
          — copies SHOWSTMT_TYPE and arg_values into SHOWSTMT_SCAN_ID
          — sets cursor = 0, ctx = NULL
start : showstmt_start_scan() — materialises rows (array pattern) or opens stream
next  : showstmt_next_scan() — returns one row per call; clears previous row's values
close : showstmt_end_scan() — frees ctx
```

## Related

- [[components/scan-manager|scan-manager]] — dispatches `S_SHOWSTMT_SCAN` to this module; holds `SHOWSTMT_SCAN_ID` in `SCAN_ID.s.stsid`
- [[components/show-meta|show-meta]] — `SHOWSTMT_METADATA` registry: DBA-only flag, column type list, argument types; semantic check hooks
- [[components/query-executor|query-executor]] — generates `S_SHOWSTMT_SCAN` access specs for SHOW statements
- [[components/btree|btree]] — `btree_index_start_scan` / `btree_index_next_scan` streaming implementation
- [[components/heap-file|heap-file]] — `heap_header_capacity_start_scan` implementation
- [[components/log-manager|log-manager]] — `log_active_log_header_start_scan` implementation
- [[components/page-buffer|page-buffer]] — `pgbuf_start_scan` implementation
