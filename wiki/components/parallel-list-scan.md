---
address: c-000003
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_scan/px_scan_input_handler_list.{cpp,hpp}, px_scan_slot_iterator_list.{cpp,hpp}"
status: branch-wip
purpose: "Parallel scan input handler + slot iterator for SCAN_TYPE::LIST: static slice partitioning of a QFILE_LIST_ID's sectors, lazy CAS-claimed membuf ownership, per-page tfile tracking for dependent-list correctness"
key_files:
  - "px_scan_input_handler_list.hpp (worker_slice struct, thread_local state, public init/initialize/get_next_page/finalize)"
  - "px_scan_input_handler_list.cpp (init_on_main slicing, initialize CAS-claim, get_next_page_with_fix two-phase loop)"
  - "px_scan_slot_iterator_list.hpp (per-worker iterator state — no thread-locals)"
  - "px_scan_slot_iterator_list.cpp (set_page tuple-count snapshot, next_qualified_slot_with_peek inner loop, overflow chain handoff to qfile_get_tuple)"
public_api:
  - "input_handler_list::init_on_main(thread_p, list_id, parallelism)"
  - "input_handler_list::initialize(thread_p, hfid, scan_id) — per-worker slice claim + membuf CAS"
  - "input_handler_list::get_next_page_with_fix(thread_p, out_page, out_tfile)"
  - "input_handler_list::finalize(thread_p)"
  - "slot_iterator_list::initialize(thread_p, scan_id, vd)"
  - "slot_iterator_list::set_page(thread_p, page, tfile)"
  - "slot_iterator_list::next_qualified_slot_with_peek(thread_p)"
  - "slot_iterator_list::finalize(thread_p)"
tags:
  - component
  - cubrid
  - parallel
  - parallel-scan
  - parallel-list-scan
  - list-file
  - branch-wip
  - CBRD-26722
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]]"
  - "[[components/parallel-hash-join-task-manager|parallel-hash-join-task-manager]]"
  - "[[components/list-file|list-file]]"
  - "[[components/file-manager|file-manager]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[flows/parallel-list-scan-open|parallel-list-scan-open]]"
  - "[[prs/PR-7062-parallel-scan-all-types|PR #7062]]"
  - "[[sources/2026-04-28-tfile-role-analysis|tfile role analysis]]"
created: 2026-04-29
updated: 2026-04-29
---

# `parallel_scan::input_handler_list` / `slot_iterator_list` — Parallel List Scan

> [!warning] Branch-WIP, not in baseline
> All file paths and line numbers on this page reflect branch `parallel_scan_all` (head `0f8a107bb`, [CBRD-26722], [[prs/PR-7062-parallel-scan-all-types|PR #7062]] OPEN). The `src/query/parallel/px_scan/` directory does **not** exist in baseline `0be6cdf6` — the heap-only predecessor `src/query/parallel/px_heap_scan/` (see [[components/parallel-heap-scan-input-handler]]) is what's on disk at baseline. Treat this page as a forward-looking design capture; it will be reconciled into the canonical `parallel-scan-input-handler-list` page when PR #7062 merges (see PR-7062 Reconciliation Plan).

The list-scan variant of the generalised `parallel_scan::manager<RESULT_TYPE, SCAN_TYPE>` template (`SCAN_TYPE::LIST`). Its job: turn a single `QFILE_LIST_ID` into N parallel readers, each holding a private slice of disk sectors plus an at-most-one-CAS-winner claim on the in-memory membuf pages.

Sibling variants in the same template family: `input_handler_heap` (renamed from `input_handler_ftabs`, see [[components/parallel-heap-scan-input-handler]]), `input_handler_index` (mutex-guarded leaf cursor).

## Pipeline

```
manager (px_scan.cpp)
    │ open() → init_on_main: collect sectors + carve N slices
    │
    ├── task<RT, LIST> per worker
    │     │
    │     ├── input_handler_list::initialize        ← claim slice idx + CAS for membuf
    │     │
    │     ├── loop:
    │     │     get_next_page_with_fix() ──► PAGE_PTR + per-page QMGR_TEMP_FILE *
    │     │            │
    │     │            ├── Phase 1 (CAS winner only): drain membuf pages
    │     │            └── Phase 2 (everyone): walk per-sector 64-page bitmap
    │     │     ↓
    │     │     slot_iterator_list::set_page  ──► snapshot tuple_count
    │     │            ↓
    │     │     while next_qualified_slot_with_peek == S_SUCCESS:
    │     │            handler.write(...)        ← result_handler does dispatch
    │     │
    │     └── finalize ── unfix last page, free tplrec scratch
    │
    └── join_jobs / aggregate (manager teardown)
```

## Static slice partitioning (no work-stealing)

`init_on_main` runs once on the main thread before any worker spawns (`px_scan_input_handler_list.cpp:44-90`). It:

1. `qfile_collect_list_sector_info (thread_p, list_id, &m_sector_info)` — reuses the helper added in [[prs/PR-6981-parallel-hash-join-sector-split|PR #6981]] (see [[components/list-file]]'s "Sector-based page distribution" section). Walks the `dependent_list_id` chain into a flat `(sectors[], tfiles[], membuf_tfile)` triple.
2. Carves `[0, sector_cnt)` into `parallelism` contiguous integer slices: `worker_slice {start, end, iter}`. Remainder distributed by giving the first `rem` slices one extra sector each.
3. `m_worker_slice_idx.store(0)` and `m_membuf_claimed.store(false)` — the two atomics workers race on at `initialize()` time.

```cpp
// px_scan_input_handler_list.hpp:41-46
struct worker_slice {
  int start;
  int end;
  int iter;   // private cursor in [start, end)
};
```

> [!key-insight] Static partition — no fault tolerance
> Each worker calls `m_worker_slice_idx.fetch_add(1)` exactly once in `initialize()` (`.cpp:95`), captures a pointer to its `m_worker_slices[idx]`, and iterates `[start, end)` privately. **There is no shared work-counter, no work-stealing, no rebalancing.** A worker that fails between `initialize()` and the first `get_next_page_with_fix()` strands its slice's sectors permanently — they never get re-claimed. Contrast with [[components/parallel-hash-join-task-manager|parallel hash join's split_task]] which uses `next_sector_index.fetch_add(1)` per *sector* (dynamic), so surviving workers absorb a dropped worker's residual sectors.

> [!gap] Fault-tolerance gap
> If a list-scan worker dies mid-slice, its sectors are silently dropped → row miss. The current correctness story relies on workers not failing after `initialize()` succeeds. Worth porting the hash-join dynamic-stealing pattern over if list scans are extended to long-running queries where worker death becomes likely.

## Lazy membuf claim — CAS, not idx-bound

```cpp
// px_scan_input_handler_list.cpp:107-117
m_tl_is_membuf_worker = false;
if (m_sector_info.membuf_tfile != nullptr)
  {
    bool expected = false;
    if (m_membuf_claimed.compare_exchange_strong (expected, true,
                                                  std::memory_order_acq_rel))
      {
        m_tl_is_membuf_worker = true;
      }
  }
```

The first **live** worker to enter `initialize()` wins the membuf via CAS. This was decoupled from slice index in commit `0f8a107bb` — earlier code bound the membuf to slice idx 0 (`m_tl_is_membuf_worker = (idx == 0 && m_has_membuf)`), which left the membuf orphaned if the idx-0 worker failed before `loop()`. Post-decoupling, a failed idx-0 worker is harmless — the next worker through `initialize()` claims the membuf instead.

> [!stale] Stale comment in source
> `px_scan_input_handler_list.cpp:134` still reads `/* Phase 1: worker 0 drains membuf pages */`. Incorrect post-`0f8a107bb` — any worker can be the membuf drainer. Cleanup candidate.

> [!key-insight] Eager claim narrows the orphan window
> List scan claims the membuf **eagerly** in `initialize()`. [[components/parallel-hash-join-task-manager|Parallel hash join's `split_task`]] claims **lazily** inside `get_next_page` (cf. `px_hash_join_task_manager.cpp:629-640`). The eager claim narrows the window where a worker could fail after taking the slice but before draining the membuf — at the cost of one CAS per worker even when the list is disk-only. Both patterns are correct; the trade-off is "extra CAS in disk-only case" vs. "wider orphan window if claimer crashes between CAS and first read."

## Two-phase page fetch (`get_next_page_with_fix`)

`px_scan_input_handler_list.cpp:122-192`. The CAS winner runs Phase 1 + Phase 2; everyone else runs Phase 2 only.

### Phase 1 — membuf drain (CAS winner only)

```cpp
if (m_tl_is_membuf_worker
    && m_sector_info.membuf_tfile != nullptr
    && m_tl_membuf_pageid <= m_sector_info.membuf_tfile->membuf_last)
  {
    vpid.volid  = NULL_VOLID;
    vpid.pageid = m_tl_membuf_pageid++;
    m_tl_current_tfile = m_sector_info.membuf_tfile;
  }
```

Sequential pageid scan from 0 to `membuf_last`, `volid = NULL_VOLID` flags the page as memory-resident to `qmgr_get_old_page_read_only`. The membuf tfile is `sector_info->membuf_tfile`, which by `qfile_collect_list_sector_info` contract is **always the base list_id's tfile** (membuf is exclusive to the head of a `dependent_list_id` chain — see [[components/list-file]]).

### Phase 2 — sector bitmap walk (everyone)

When the bitmap is empty, refill from the next sector in the worker's slice:

```cpp
if (m_tl_bitmap == 0)
  {
    int sidx = m_tl_slice->iter++;
    if (sidx >= m_tl_slice->end) return S_END;
    m_tl_vsid          = m_sector_info.sectors[sidx].vsid;
    m_tl_bitmap        = m_sector_info.sectors[sidx].page_bitmap;
    m_tl_current_tfile = (QMGR_TEMP_FILE *) m_sector_info.tfiles[sidx];
    if (m_tl_bitmap == 0) continue;  // defensive: empty sector
  }
```

Page extraction via `__builtin_ctzll` of the lowest set bit, then `m_tl_bitmap &= m_tl_bitmap - 1` clears it.

> [!key-insight] Per-sector tfile tracking
> Like [[components/parallel-hash-join-task-manager|parallel hash join's split_task]], list scan reads `m_sector_info.tfiles[sidx]` per sector, not `list_id->tfile_vfid`. Required for correctness when `dependent_list_id` chain is non-empty: dependent-list pages must be released against the dependent's `QMGR_TEMP_FILE *`, not the base list's. The membuf has its own tfile (`m_sector_info.membuf_tfile`) carried separately in Phase 1.

### Overflow continuation skip

```cpp
if (QFILE_GET_TUPLE_COUNT (page_p) == QFILE_OVERFLOW_TUPLE_COUNT_FLAG)
  {
    qmgr_free_old_page (thread_p, page_p, m_tl_current_tfile);
    continue;
  }
```

Same mechanism as [[components/parallel-hash-join-task-manager|hash join's split_task]] (see [[components/list-file|list-file]] § `QFILE_OVERFLOW_TUPLE_COUNT_FLAG`). Continuation pages share the start page's sector bitmap because of `qfile_allocate_new_ovf_page`; the start-page owner walks the chain via `qfile_get_tuple`, so peer workers must drop continuation pages on encounter.

## Slot iterator — silent-skip sentinels

`slot_iterator_list` is the per-worker tuple-level cursor. It snapshots `tuple_count` at `set_page` time (`px_scan_slot_iterator_list.cpp:108`):

```cpp
m_tuple_count = QFILE_GET_TUPLE_COUNT (m_curr_pgptr);
```

and gates the inner loop by `while (m_curr_tplno < m_tuple_count)` (`.cpp:118`).

> [!key-insight] TWO header conditions silently skip a page
> The cursor pipeline silently drops a page on either of:
>
> 1. `tuple_count == QFILE_OVERFLOW_TUPLE_COUNT_FLAG (-2)` — filtered at `get_next_page_with_fix` (`.cpp:182-186`). **Intended** — overflow continuation pages are consumed by the start-page owner.
> 2. `tuple_count == 0` — falls through `set_page` cleanly, but `next_qualified_slot_with_peek` returns `S_END` immediately because the inner-loop guard `m_curr_tplno < m_tuple_count` is false. **Unintended consequence** — if a writer is mid-allocation (between `qfile_allocate_new_page` initialising the header to 0 via `QFILE_PAGE_HEADER_INITIALIZER` and `qfile_allocate_new_page_if_need` doing `QFILE_PUT_TUPLE_COUNT (*page_p, ... + 1)` at `list_file.c:1581`), and `file_get_all_data_sectors` runs in the same window, the page appears in the bitmap with `tuple_count == 0` — silent miss for that page.
>
> The `run_jobs()` join barrier in [[components/parallel-query-executor]] (`px_query_executor.cpp:115-175`) is the **only** thing keeping this race closed for the parallel-aptr → parallel-list-scan pattern. Any path that opens a parallel list scan against a list still being written would expose the race. See [[flows/parallel-list-scan-open|parallel-list-scan-open]] for the happens-before chain.

## Overflow handling in `next_qualified_slot_with_peek`

`px_scan_slot_iterator_list.cpp:113-188`. Computes `has_overflow_page = (QFILE_GET_OVERFLOW_PAGE_ID(m_curr_pgptr) != NULL_PAGEID)` once at function entry. For overflow start pages, calls `qfile_get_tuple (thread_p, m_curr_pgptr, m_curr_tpl, &m_tplrec, m_list_id)` — which walks the `OVERFLOW_VPID` chain via repeated `qmgr_get_old_page (..., list_id_p->tfile_vfid)` (see `list_file.c:4663-4726`).

> [!gap] Dependent-list overflow uses base tfile
> `qfile_get_tuple` uses `m_list_id->tfile_vfid` (i.e. the base list's tfile) for overflow chain `qmgr_get_old_page` calls — even though `set_page` carries a per-page tfile in `m_curr_tfile`. For our common case (single list, no `dependent_list_id`) base tfile == per-page tfile, so this is correct. For lists where the start page lives in a *dependent* list_id and its overflow continuation also lives in the dependent leg, this would fix the wrong tfile.
>
> Verify before extending parallel list scan to `dependent_list_id` chains that produce long tuples in the dependent leg. Most call sites today (BUILDLIST_PROC inner subqueries materialised via aptr) don't trigger this combination, but the gap is real.

## Thread-local state (per-worker singletons)

```cpp
// px_scan_input_handler_list.hpp — class-static thread_local members
thread_local input_handler_list::worker_slice *input_handler_list::m_tl_slice           = nullptr;
thread_local UINT64                            input_handler_list::m_tl_bitmap          = 0;
thread_local VSID                              input_handler_list::m_tl_vsid            = {NULL_SECTID, NULL_VOLID};
thread_local QMGR_TEMP_FILE                   *input_handler_list::m_tl_current_tfile  = nullptr;
thread_local bool                              input_handler_list::m_tl_is_membuf_worker = false;
thread_local int                               input_handler_list::m_tl_membuf_pageid   = 0;
```

Same convention introduced for parallel heap scan in [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911]] and reused by every parallel-scan input handler since: at most one parallel scan runs per worker thread at a time (subsequent scans in NL-join contexts run as `scan_ptr`), so a class-level `thread_local` singleton is safe.

## Differences from the heap variant

| Aspect | `input_handler_heap` (baseline) | `input_handler_list` (this page) |
|---|---|---|
| Distribution unit | `ftab_set::split` over `FILE_FTAB_COLLECTOR` | bespoke `worker_slice {start, end, iter}` |
| Distribution model | dynamic via `m_splited_ftab_set_idx.fetch_add(1)` per group | **static** integer slice — claim once, iterate privately |
| Membuf | n/a (heap files have no membuf) | lazy CAS-claim by first live worker |
| Per-page tfile | always `hfid`-derived (single backing file) | per-sector lookup (`m_sector_info.tfiles[sidx]`) |
| Dependent chain | n/a | walks `dependent_list_id`, concatenates sectors |
| Overflow handling | n/a (heap pages don't have QFILE overflow) | skip on `QFILE_OVERFLOW_TUPLE_COUNT_FLAG` |

## Differences from `parallel_hash_join::split_task` (the closest sibling)

Both consume `qfile_collect_list_sector_info`'s output. Differences:

| Aspect | `split_task` (PR #6981, merged) | `input_handler_list` (PR #7062, branch) |
|---|---|---|
| Sector distribution | dynamic — `next_sector_index.fetch_add(1)` per sector | **static** — one slice claim per worker |
| Membuf claim timing | lazy — inside `get_next_page` on first need | eager — in `initialize()` before any I/O |
| Membuf claim location | `px_hash_join_task_manager.cpp:629-640` | `px_scan_input_handler_list.cpp:107-117` |
| Fault tolerance | survives worker death (sectors picked up by others) | none — dropped slice is permanently lost |
| Tfile ownership tracking | per-thread `m_current_tfile` recorded in `get_next_page` | thread-local `m_tl_current_tfile` carried by handler, returned to caller |

The two designs converged independently on the same primitive (`qfile_collect_list_sector_info`) but made different trade-offs around dynamic-vs-static and eager-vs-lazy. Both designs respect [[sources/2026-04-28-tfile-role-analysis|the tfile-role contract]]: per-page tfile is required for free-time discrimination on dependent-list pages.

## Constraints

- **Threading**: `init_on_main` runs single-threaded on main; `initialize`/`get_next_page_with_fix`/`finalize` run per-worker.
- **Build-mode**: SERVER_MODE + SA_MODE only.
- **Lifetime**: handler is `db_private_alloc`'d + placement-new constructed by `manager::open`; explicit destructor + `db_private_free` on teardown (see [[prs/PR-7062-parallel-scan-all-types]] for the placement-new+free triplet contract).
- **Snapshot mode**: `XASL_SNAPSHOT × LIST` is **blocked by the checker** (`px_scan_checker.cpp`); only `MERGEABLE_LIST` and `BUILDVALUE_OPT` modes use this handler.

## Related

- Parent (template manager): [[components/parallel-heap-scan|parallel-heap-scan]] (will be renamed to `parallel-scan` on PR #7062 merge)
- Sibling variant: [[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]] (heap)
- Companion flow: [[flows/parallel-list-scan-open|parallel-list-scan-open]]
- Underlying primitive: [[components/list-file|list-file]] § "Sector-based page distribution"
- Fault-tolerance contrast: [[components/parallel-hash-join-task-manager|parallel-hash-join-task-manager]] § "Page Cursor — Lock-free sector bitmap walk"
- tfile contract: [[sources/2026-04-28-tfile-role-analysis|tfile role analysis]]
- Tracking PR: [[prs/PR-7062-parallel-scan-all-types|PR #7062]]
- Jira: CBRD-26722
