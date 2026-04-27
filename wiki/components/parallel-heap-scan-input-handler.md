---
status: developing
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_ftab_set.hpp"
path_impl: "src/query/parallel/px_heap_scan/px_heap_scan_input_handler_ftabs.hpp"
path_impl2: "src/query/parallel/px_heap_scan/px_heap_scan_input_handler_ftabs.cpp"
tags:
  - component
  - cubrid
  - parallel
  - query
  - heap-scan
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/storage|storage]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan_input_handler` — Page-Set Distribution to Workers

Two classes manage how heap pages are divided and delivered to parallel worker threads: `ftab_set` (a value type representing a slice of the heap file's sector table) and `input_handler_ftabs` (the coordinator that splits and distributes those slices).

## Class / Function Inventory

### `ftab_set` (header-only, `px_ftab_set.hpp`)

> [!update] PR #7011 (merge `cc563c7f`) — file moved + namespace migrated
> Header relocated from `src/query/parallel/px_heap_scan/px_heap_scan_ftab_set.hpp` to `src/query/parallel/px_ftab_set.hpp`. Class moved from `parallel_heap_scan` to `parallel_query` namespace so both heap-scan and parallel index-build (`external_sort.c::sort_start_parallelism` for `SORT_INDEX_LEAF`) can share it. `parallel_heap_scan::ftab_set` is preserved as a `using`-alias inside `px_heap_scan_input_handler_ftabs.hpp` to keep existing heap-scan call-sites compiling unchanged. CMakeLists moves the header from `PARALLEL_HEAP_SCAN_HEADERS` to `PARALLEL_QUERY_HEADERS`.

| Method | Description |
|--------|-------------|
| `ftab_set()` | Default ctor; empty vector, iterator = 0 |
| `~ftab_set()` | Dtor; clears `m_ftab_set` (added by PR #7011) |
| `ftab_set(const ftab_set&)` / `operator=(const ftab_set&)` | Copy ctor + copy assignment (added by PR #7011) |
| `ftab_set(ftab_set&&)` / `operator=(ftab_set&&)` | Move ctor + move assignment (added by PR #7011) |
| `convert(FILE_FTAB_COLLECTOR*)` | Copies `partsect_ftab[0..nsects]` into `m_ftab_set` |
| `split(int n_sets) -> vector<ftab_set>` | Divides sectors among `n_sets` slices; remainder sectors go to first slices |
| `append(const ftab_set &other)` | Concatenates `m_ftab_set` lists (added by PR #7011) |
| `move_from(ftab_set &other)` | `std::move` content + reset other's iterator (added by PR #7011) |
| `size() const -> size_t` | Exposes underlying vector size (added by PR #7011) |
| `get_next() -> FILE_PARTIAL_SECTOR` | Returns next sector and advances iterator; returns `FILE_PARTIAL_SECTOR_INITIALIZER` at end |
| `clear()` | Clears vector and resets iterator |

**Fields**: `m_ftab_set` (`vector<FILE_PARTIAL_SECTOR>`), `iterator` (`size_t`).

The new copy/move semantics + `append`/`move_from`/`size` support the "split into per-worker `ftab_set`s and pass by value/move into per-worker `SORT_ARGS`" pattern used by parallel CREATE INDEX (see [[components/btree#parallel-index-build-sort_index_leaf]]).

### `input_handler_ftabs` (`px_heap_scan_input_handler_ftabs.hpp/.cpp`)

| Method | Description |
|--------|-------------|
| `input_handler_ftabs(interrupt*, err_messages*)` | Ctor; stores interrupt/error pointers |
| `init_on_main(thread_p, hfid, parallelism)` | Main-thread init: calls `file_get_all_data_sectors`, converts to `ftab_set`, splits into `parallelism` slices, stores in `m_splited_ftab_set` |
| `initialize(thread_p, hfid*, scan_id*)` | Per-worker init: assigns a slice via atomic `fetch_add`, sets up TLS fields |
| `get_next_vpid_with_fix(thread_p, vpid*)` | Returns next valid heap page VPID with buffer fix; handles deallocated pages gracefully |
| `finalize(thread_p)` | Per-worker teardown: unfix page watchers, reset TLS pointers |

**TLS (thread_local static) fields**:

| TLS Field | Type | Role |
|-----------|------|------|
| `m_tl_vpid` | `VPID` | Current VPID within sector |
| `m_tl_scan_cache` | `HEAP_SCANCACHE*` | Points to worker's scan cache from SCAN_ID |
| `m_tl_old_page_watcher` | `PGBUF_WATCHER` | Previous page watcher for ordered unfix |
| `m_tl_ftab_set` | `ftab_set*` | Pointer to this worker's assigned ftab slice |
| `m_tl_pgoffset` | `size_t` | Offset within current sector (0..DISK_SECTOR_NPAGES) |
| `m_tl_ftab` | `FILE_PARTIAL_SECTOR` | Current sector being iterated |

**Shared fields**:

| Field | Type | Role |
|-------|------|------|
| `m_ftab_set` | `ftab_set` | Master set (cleared after split) |
| `m_splited_ftab_set` | `vector<ftab_set>` | One slice per worker |
| `m_splited_ftab_set_idx` | `atomic_int` | Next unclaimed slice index (fetch_add) |
| `m_hfid` | `HFID` | Heap file ID (used to skip header page) |
| `m_interrupt_p` | `interrupt*` | Shared interrupt signal |
| `m_err_messages_p` | `err_messages_with_lock*` | Error message propagation |

## Execution Path

### Main Thread (`init_on_main`)

```
init_on_main(thread_p, hfid, parallelism)
  │
  ├─ file_get_all_data_sectors(thread_p, &hfid.vfid, &collector)
  │    └─ returns FILE_FTAB_COLLECTOR with partsect_ftab[nsects]
  ├─ m_ftab_set.convert(&collector)
  ├─ m_splited_ftab_set = m_ftab_set.split(parallelism)
  │    └─ round-robin: slice i gets ⌊nsects/n⌋ + (1 if i < remainder)
  ├─ m_splited_ftab_set_idx.store(0)
  └─ m_ftab_set.clear()   (free master copy)
       db_private_free(collector.partsect_ftab)
```

### Worker Thread (`initialize`)

```
initialize(thread_p, hfid, scan_id)
  │
  ├─ m_tl_scan_cache = &scan_id->s.hsid.scan_cache
  ├─ PGBUF_INIT_WATCHER(&m_tl_old_page_watcher, PGBUF_ORDERED_HEAP_NORMAL, hfid)
  ├─ idx = m_splited_ftab_set_idx.fetch_add(1)
  └─ m_tl_ftab_set = &m_splited_ftab_set[idx]
```

### Worker Thread (`get_next_vpid_with_fix`)

```
while !found:
  if m_tl_vpid is null:
    m_tl_ftab = m_tl_ftab_set->get_next()
    if sector is null → unfix old watcher → S_END
    compute first VPID from sector: SECTOR_FIRST_PAGEID(vsid.sectid)
    skip header page if VPID == hfid.vfid.fileid

  for m_tl_pgoffset in [0, DISK_SECTOR_NPAGES):
    if bit64_is_set(m_tl_ftab.page_bitmap, pgoffset):
      pgbuf_replace_watcher(old_watcher → scan_cache.page_watcher)
      pgbuf_ordered_fix(m_tl_vpid, OLD_PAGE_MAYBE_DEALLOCATED, LATCH_READ)
      if page null and error == ER_PB_BAD_PAGEID:
        page was deallocated after bitmap was built → skip (not an error)
      else if page null:
        propagate error → interrupt → S_ERROR
      assert PAGE_HEAP
      *vpid = m_tl_vpid; advance m_tl_pgoffset; return S_SUCCESS

  VPID_SET_NULL (end of sector)
```

> [!key-insight] Bitmap-based page presence check eliminates fixed cost
> The `FILE_PARTIAL_SECTOR::page_bitmap` is a 64-bit bitmap (one bit per page in the sector). `bit64_is_set` checks in O(1) whether a page belongs to the heap file without reading the page. Only pages marked present in the bitmap are fixed — empty or unallocated slots are skipped at zero I/O cost.

> [!key-insight] Atomic fetch-add for lock-free slice assignment
> `m_splited_ftab_set_idx.fetch_add(1)` gives each worker a unique slice index without a mutex. Workers that call `initialize` in any order each receive a distinct slice. If `idx >= m_splited_ftab_set.size()` the assertion fails (more workers than slices — should not happen given `parallelism` matches).

> [!warning] Deallocated pages are silently skipped
> `ER_PB_BAD_PAGEID` is treated as a benign "page no longer exists" condition and the page is skipped with `er_clear()`. This can happen when a concurrent DDL drops a partition or truncates a table between when the bitmap was built and when the worker tries to fix the page. Only actual I/O errors propagate as `S_ERROR`.

## Constraints

- **Thread-local state**: all `m_tl_*` fields are `thread_local static` — one copy per OS thread. Correct only because each thread calls `initialize` once per task execution and `finalize` at teardown.
- **Page ordering**: `pgbuf_ordered_fix` is used with `PGBUF_ORDERED_HEAP_NORMAL` to respect the buffer pool's ordered-latch protocol and avoid deadlocks with concurrent heap operations.
- **Memory**: `m_splited_ftab_set` is a `vector` on the `input_handler` object (private heap of main thread). TLS pointers reference entries in this vector — they must not outlive the manager.
- **Build mode**: active in `SERVER_MODE` and `SA_MODE`.

## Lifecycle

```
1. main: init_on_main — collect sectors, split into slices
2. worker: initialize — claim slice, set up TLS
3. worker: [loop] get_next_vpid_with_fix — iterate pages
4. worker: finalize — unfix watchers, clear TLS pointers
5. main: manager::reset() or close() — input_handler destructor frees m_splited_ftab_set
```

## Related

- [[components/parallel-heap-scan|parallel-heap-scan]] — parent hub
- [[components/heap-file|heap-file]] — `file_get_all_data_sectors`, `heap_next_1page`, scan cache
- [[components/storage|storage]] — `pgbuf_ordered_fix`, `PGBUF_WATCHER`, `FILE_PARTIAL_SECTOR`
- [[Memory Management Conventions]] — `db_private_alloc` / `db_private_free` conventions
