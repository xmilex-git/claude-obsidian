---
type: pr
pr_number: 7062
pr_url: "https://github.com/CUBRID/cubrid/pull/7062"
repo: "CUBRID/cubrid"
state: OPEN
is_draft: false
author: "xmilex-git"
created_at:
merged_at:
closed_at:
merge_commit:
base_ref: "develop"
head_ref: "parallel_scan_all"
base_sha: "1da6caa7d6221f4543ef69a5deba12e074e4290b"
head_sha: "e66051b9e97ead89fbc25265f758985c094e48a4"
jira: "CBRD-26722"
files_changed:
  - "CMakeLists.txt"
  - "cs/CMakeLists.txt"
  - "cubrid/CMakeLists.txt"
  - "sa/CMakeLists.txt"
  - "src/base/system_parameter.c"
  - "src/base/system_parameter.h"
  - "src/optimizer/plan_generation.c"
  - "src/parser/csql_grammar.y"
  - "src/parser/name_resolution.c"
  - "src/parser/parse_tree.h"
  - "src/parser/parse_tree_cl.c"
  - "src/parser/scanner_support.c"
  - "src/parser/xasl_generation.c"
  - "src/query/query_dump.c"
  - "src/query/query_executor.c"
  - "src/query/scan_manager.c"
  - "src/query/scan_manager.h"
  - "src/query/xasl.h"
  - "src/storage/btree.c"
  - "src/storage/btree.h"
  - "src/query/parallel/px_heap_scan/px_heap_scan.cpp (deleted)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_trace_handler.cpp (deleted)"
  - "src/query/parallel/px_parallel.cpp"
  - "src/query/parallel/px_parallel.hpp"
  - "src/query/parallel/px_scan/px_scan.cpp (new, 2160 LOC)"
  - "src/query/parallel/px_scan/px_scan.hpp (renamed from px_heap_scan.hpp)"
  - "src/query/parallel/px_scan/px_scan_checker.cpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_checker.hpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_ftab_set.hpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_input_handler_heap.cpp (renamed from *_ftabs.cpp)"
  - "src/query/parallel/px_scan/px_scan_input_handler_heap.hpp (renamed from *_ftabs.hpp)"
  - "src/query/parallel/px_scan/px_scan_input_handler_index.cpp (new, 243 LOC)"
  - "src/query/parallel/px_scan/px_scan_input_handler_index.hpp (new)"
  - "src/query/parallel/px_scan/px_scan_input_handler_list.cpp (new, 228 LOC)"
  - "src/query/parallel/px_scan/px_scan_input_handler_list.hpp (new)"
  - "src/query/parallel/px_scan/px_scan_join_info.cpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_join_info.hpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_result_handler.cpp (renamed + extended)"
  - "src/query/parallel/px_scan/px_scan_result_handler.hpp (renamed + extended)"
  - "src/query/parallel/px_scan/px_scan_result_type.hpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator.cpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator.hpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator_index.cpp (new, 1026 LOC)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator_index.hpp (new)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator_list.cpp (new, 200 LOC)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator_list.hpp (new)"
  - "src/query/parallel/px_scan/px_scan_task.cpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_task.hpp (renamed)"
  - "src/query/parallel/px_scan/px_scan_trace_handler.cpp (new, 434 LOC)"
  - "src/query/parallel/px_scan/px_scan_trace_handler.hpp"
  - "src/query/parallel/px_scan/px_scan_type.hpp (new)"
  - "src/query/parallel/px_scan/px_scan_type_enum.hpp (new)"
related_components:
  - "[[components/parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-input-handler]]"
  - "[[components/parallel-heap-scan-result-handler]]"
  - "[[components/parallel-heap-scan-slot-iterator]]"
  - "[[components/parallel-heap-scan-task]]"
  - "[[components/parallel-heap-scan-join-info]]"
  - "[[components/parallel-heap-scan-support]]"
  - "[[components/parallel-query]]"
  - "[[components/parallel-query-checker]]"
  - "[[components/parallel-query-executor]]"
  - "[[components/scan-manager]]"
  - "[[components/xasl]]"
  - "[[components/btree]]"
  - "[[components/parser]]"
  - "[[components/xasl-generation]]"
  - "[[components/query-dump]]"
related_sources:
  - "[[sources/cubrid-src-query-parallel]]"
  - "[[sources/cubrid-src-query]]"
  - "[[sources/cubrid-src-parser]]"
  - "[[sources/cubrid-src-storage]]"
ingest_case: "open"
triggered_baseline_bump: false
baseline_before: "175442fc858bd0075165729756745be6f8928036"
baseline_after: "175442fc858bd0075165729756745be6f8928036"
reconciliation_applied: false
reconciliation_applied_at:
incidental_enhancements_count: 1
tags:
  - pr
  - cubrid
  - parallel-query
  - parallel-scan
  - index-scan
  - list-scan
  - heap-scan
  - refactor
  - performance
created: 2026-04-24
updated: 2026-04-24
status: open
---

# PR #7062 — Expand parallel heap scan to parallel scan (index, heap, list) (CBRD-26722)

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `OPEN` (non-draft) · **Author:** `@xmilex-git` · **Jira:** CBRD-26722
> **Base → Head:** `develop` (`1da6caa7`) → `parallel_scan_all` (`e66051b9`)
> **Stats:** 52 files changed, 77 commits, 26 reviews, 37 review-thread comments, ≈ +6,190 / −1,893 LOC

> [!note] Ingest classification: open
> PR is open, not merged. **No component-page edits for PR-induced changes.** A full Reconciliation Plan is written below, executable later when the user says "apply reconciliation for PR #7062". Incidental wiki enhancements from baseline analysis WERE applied — see the dedicated section.

> [!tip] Authoritative design reference
> The author attached an external code-review document (`pr_7062_code_review.md`) to the PR, 358 lines, covering scan types, design decisions per scan type, checker rules, system parameters, result-handler matrix, and safety measures. Much of this PR page synthesises from that document — when something feels dense here, the original is the source of truth: https://github.com/user-attachments/files/26920618/pr_7062_code_review.md

## Summary

Generalises the parallel-heap-scan infrastructure into a scan-type-agnostic `parallel_scan::manager<RESULT_TYPE, SCAN_TYPE>` template that supports **heap**, **list**, and **index** scans uniformly. The existing `src/query/parallel/px_heap_scan/` directory and `parallel_heap_scan` namespace are retired; replaced by `src/query/parallel/px_scan/` and the `parallel_scan` namespace. Three new scan-type variants land: `input_handler_heap` (renamed from `input_handler_ftabs`), `input_handler_list`, `input_handler_index`, each with a matching `slot_iterator_*`. Result handling, worker management, interrupt propagation, and tracing are shared across all three scan types via `scan_traits<ST>` template specialisation.

New enum values in `scan_manager.h`: `S_PARALLEL_LIST_SCAN`, `S_PARALLEL_INDEX_SCAN` (previously only `S_PARALLEL_HEAP_SCAN`). New `SCAN_ID` union members: `pllsid_parallel`, `pisid` (co-exist with the single-threaded `llsid`, `isid` for fallback).

## Motivation

Heap scan was the only parallel-capable scan. Large list-file materialisations (complex joins with intermediate results) and large index range scans (full-range `SELECT col WHERE indexed_col BETWEEN …`) could not benefit from worker parallelism and remained single-threaded bottlenecks. Extending the parallel framework from one scan type to three gives all three heavy-I/O paths access to worker-level parallelism while keeping the user-facing hint surface unchanged (`/*+ NO_PARALLEL_SCAN */` replaces `/*+ NO_PARALLEL_HEAP_SCAN */` as a general kill switch; no new hints introduced).

## Changes

### Structural

**Namespace and directory rename (all files moved):**
- `src/query/parallel/px_heap_scan/` → `src/query/parallel/px_scan/`
- `namespace parallel_heap_scan` → `namespace parallel_scan`
- Many files renamed accordingly (e.g. `px_heap_scan_checker.cpp` → `px_scan_checker.cpp`). One meaningful semantic rename: `input_handler_ftabs` → `input_handler_heap` — "ftabs" referenced the internal file-table-allocation-set data structure and was too narrow; "heap" puts it on the same conceptual layer as the new `_list` and `_index` variants.

**Deleted (content now in px_scan.cpp):**
- `src/query/parallel/px_heap_scan/px_heap_scan.cpp` (1116 LOC — entire heap-specific manager class)
- `src/query/parallel/px_heap_scan/px_heap_scan_trace_handler.cpp` (231 LOC — subsumed by unified trace_handler)

**New files (non-rename):**
- `src/query/parallel/px_scan/px_scan.cpp` (2160 LOC) — unified manager/task/wrapper implementation; consolidates the heap-specific code + generalises it
- `src/query/parallel/px_scan/px_scan_input_handler_index.cpp/.hpp` (243 + 83 LOC) — mutex-guarded leaf-page cursor
- `src/query/parallel/px_scan/px_scan_input_handler_list.cpp/.hpp` (228 + 84 LOC) — sector pre-split on temp file
- `src/query/parallel/px_scan/px_scan_slot_iterator_index.cpp/.hpp` (1026 + 102 LOC) — by far the biggest new file; key-level iteration inside a locked leaf page + heap-lookup logic
- `src/query/parallel/px_scan/px_scan_slot_iterator_list.cpp/.hpp` (200 + 62 LOC) — tuple iteration within a sector range
- `src/query/parallel/px_scan/px_scan_trace_handler.cpp` (434 LOC) — unified trace handler
- `src/query/parallel/px_scan/px_scan_type.hpp` (63 LOC) — `SCAN_TYPE` enum (HEAP | LIST | INDEX) + `scan_traits<ST>` template framework
- `src/query/parallel/px_scan/px_scan_type_enum.hpp` (38 LOC) — enum helper

**Modified C sources (integration):**
- `src/query/scan_manager.c/.h` (+152 LOC) — adds `S_PARALLEL_LIST_SCAN`, `S_PARALLEL_INDEX_SCAN` cases to all dispatch points: `scan_start`, `scan_next`, `scan_reset`, `scan_end`, `scan_close`, `scan_dump`, `scan_stat`. New union members `pllsid_parallel`, `pisid`. New helper functions `scan_open_parallel_list_scan`, `scan_open_parallel_index_scan`, etc.
- `src/query/query_executor.c` (+85/−30) — parallel-scan invocation from the XASL executor; feeds aggregation domains for the `BUILDVALUE_OPT` result mode
- `src/query/query_dump.c` (+57/−3) — trace output covers the new scan types
- `src/query/xasl.h` — `ACCESS_SPEC_FLAG_NO_PARALLEL_HEAP_SCAN` renamed to `ACCESS_SPEC_FLAG_NO_PARALLEL_SCAN`; `ACCESS_SPEC_FLAG_COUNT_DISTINCT` renamed to `ACCESS_SPEC_FLAG_BUILDVALUE_OPT`
- `src/parser/csql_grammar.y`, `src/parser/parse_tree.h`, `src/parser/parse_tree_cl.c`, `src/parser/scanner_support.c`, `src/parser/name_resolution.c` — `PT_HINT_NO_PARALLEL_HEAP_SCAN` → `PT_HINT_NO_PARALLEL_SCAN`; `PT_SPEC_FLAG_NO_PARALLEL_HEAP_SCAN` → `PT_SPEC_FLAG_NO_PARALLEL_SCAN`; propagation path widened to list/index specs
- `src/parser/xasl_generation.c` (+30/−23) — propagates the renamed `ACCESS_SPEC_FLAG_NO_PARALLEL_SCAN` to list and index specs (not only heap specs as before)
- `src/optimizer/plan_generation.c` (+157/−0) — plan-generation hooks for parallel list/index costing
- `src/storage/btree.c/.h` (+62/−20) — adds helper used by `input_handler_index`'s vertical traversal and leaf-cursor advance
- `src/base/system_parameter.c/.h` — **new** parameter `parallel_scan_page_threshold` (see Behavioral)
- `src/query/parallel/px_parallel.cpp/.hpp` — namespace registration updated
- `CMakeLists.txt`, `cubrid/CMakeLists.txt`, `cs/CMakeLists.txt`, `sa/CMakeLists.txt` — source-list updates for the new `px_scan/` directory

### Behavioral

- **RESULT_TYPE × SCAN_TYPE matrix is 7 of 9 valid.** Workable combinations: `MERGEABLE_LIST × {heap, list, index}`, `BUILDVALUE_OPT × {heap, list, index}`, `XASL_SNAPSHOT × heap`. **Blocked** by the checker: `XASL_SNAPSHOT × list`, `XASL_SNAPSHOT × index`. Reason: `XASL_SNAPSHOT` is the row-by-row mode where main calls `next()` one row at a time and the scan cursor is shared between main and workers. Heap cursors are page-level with disjoint partitions and are safe; list/index parallel state (`llsid` / `isid` union reinterpretations) produces row-miss-or-duplicate hazards when snapshotted mid-iteration. The final cleanup commit `c28c5945a` removed the dead code paths that assumed this combo was reachable.

- **Index scan — mutex-guarded leaf chain.** Workers share a leaf cursor `VPID m_current_leaf_vpid` protected by `std::mutex m_leaf_mutex` (`px_scan_input_handler_index.hpp:71-73`). Each worker acquires the mutex → reads one leaf → advances cursor to `hdr->next_vpid` (or `prev_vpid` under `use_desc_index`) → releases the mutex → processes all keys in that leaf independently. The mutex is contended **once per thousands of key-processing operations**, so amortises to noise on large indexes. Small indexes suffer; hence the `parallel_scan_page_threshold` gate (see next bullet). The key-range-partition alternative was considered and rejected (skew sensitivity + setup cost of bounding-key lookups).

- **Index scan — vertical traversal is inlined, not via `btree_find_first()`.** Root-to-start-leaf descent happens on main thread in `init_on_main` (`px_scan_input_handler_index.cpp:51-173`) via `spage_get_record` + `OR_GET_INT` / `OR_GET_SHORT` raw parsing of non-leaf records. B+tree internal record format becomes a **synchronisation-sensitive contract** — any format change in storage will force an edit in the parallel-scan input handler.

- **List scan — temp-file sectors pre-split.** On open, `file_get_all_data_sectors` collects the disk-resident sectors of the backing `QFILE_LIST_ID` temp file, and the `ftab_set` is split evenly across workers. Workers then iterate their owned sector range without cross-thread coordination. **Membuf fallback:** if the list is small enough to live entirely in memory (`m_has_membuf == true`, no `temp_vfid`), sector partition is impossible and the scan falls back to single-threaded. **Dependent lists:** when the list references another (`dependent_list_id`), both temp files are merged into one collector so no sectors leak.

- **New system parameter `parallel_scan_page_threshold`** (`PRM_ID_PARALLEL_SCAN_PAGE_THRESHOLD`): default `2048`, min `2048`, max `INT_MAX`, flags `PRM_FOR_SERVER | PRM_HIDDEN`. Kills parallelisation when the scan target has fewer than 2048 pages. History: originally introduced as `parallel_index_scan_page_threshold` with default `32` (commit `8827dd968`), raised to `2048` after measurement showed leaf-mutex contention outweighs parallelism on small indexes (`e17db63eb`), then renamed to `parallel_scan_page_threshold` and unified across all three scan types (`9551c8356`).

- **Hint grammar UNCHANGED from the user's perspective.** `/*+ NO_PARALLEL_SCAN */` replaces the identifier `/*+ NO_PARALLEL_HEAP_SCAN */` (old hint name no longer accepted), but the replacement one line now disables heap + list + index parallel scans globally for the statement. No other new hints introduced.

- **Index-scan promotion is deferred to `scan_start_scan`.** `scan_open_parallel_index_scan` does NOT open the manager eagerly; it leaves `scan_id` in a pending hybrid state where both `isid` (single-thread) and `pisid` (parallel) coexist in the union. `scan_start_scan` attempts promotion to parallel and, on failure, falls back to `S_INDX_SCAN` cleanly without corrupting the union. This handles cases where the preparation-time decision is invalidated by a runtime condition (reserved workers unavailable, system params flipped mid-prepare, etc.).

- **Per-worker stats are split by partition in trace output.** `query_dump.c` now emits per-worker partition-statistics lines separately for heap/list/index, not a merged rollup — makes EXPLAIN-time analysis of skew tractable.

- **Enum/flag renames (caller surface):**
  - `RESULT_TYPE::COUNT_DISTINCT` → `RESULT_TYPE::BUILDVALUE_OPT`
  - `ACCESS_SPEC_FLAG_COUNT_DISTINCT` → `ACCESS_SPEC_FLAG_BUILDVALUE_OPT`
  - `ACCESS_SPEC_FLAG_NO_PARALLEL_HEAP_SCAN` → `ACCESS_SPEC_FLAG_NO_PARALLEL_SCAN`
  - `PT_HINT_NO_PARALLEL_HEAP_SCAN` → `PT_HINT_NO_PARALLEL_SCAN`
  - `PT_SPEC_FLAG_NO_PARALLEL_HEAP_SCAN` → `PT_SPEC_FLAG_NO_PARALLEL_SCAN`
  - `scan_open_parallel_heap_scan` retained; **new** `scan_open_parallel_list_scan`, `scan_open_parallel_index_scan` (and start/next/reset/end/close wrappers).

### Checker — significantly expanded parallelisation blockers

**Index-scan-specific (all unconditional NO_PARALLEL on the INDEX spec):**

| Condition | Why | Location |
|---|---|---|
| `ACCESS_SPEC_FLAG_ONLY_MIN_MAX_SCAN` (MRO) | Reads only first/last key then exits — parallel has no win | `px_scan_checker.cpp:423-428` |
| `indexptr->use_iss` (Index Skip Scan) or `ils_prefix_len > 0` (Loose Scan) | Cursor-control logic too complex for shared-cursor model | `:433-436` |
| `key_info.is_user_given_keylimit` (KEYLIMIT) | Limit depends on global key order | `:440-443` |
| `orderby_skip` / `groupby_skip` / `orderby_desc` / `groupby_desc` | Sort-elision relies on cursor order; breaks under parallel consumption | `:449-453` |
| `use_desc_index` (descending scan) | Relatively untested; conservative disable | `:459-462` |
| `is_filtered_index(...)` (partial index `CREATE INDEX … WHERE …`) | Leaf chain has implicit filter; detection via schema_manager BTID match (function indexes pass through) | `:470-473` |

**XASL-tree-level propagation (cross-scan):**

| Condition | Effect | Location |
|---|---|---|
| `instnum_pred` / `instnum_val` (ROWNUM), `XASL_ANALYTIC_SKIP_SORT`, `XASL_ANALYTIC_USES_LIMIT_OPT` | INDEX spec NO_PARALLEL only (heap/list allowed) | `:891-896` |
| `XASL_SKIP_ORDERBY_LIST` | ALL specs NO_PARALLEL (INSERT ordering) | `:891-896` |
| `MERGELIST_PROC` | Node blocked + outer/inner spec_list + aptr_list subtree's index/temp scans recursively blocked (`block_parallel_index_and_temp_in_subtree`) | `:829-841, 955-989` |
| `XASL_SNAPSHOT × LIST/INDEX` (row-by-row mode) | Forced NO_PARALLEL | `:937-948` |
| `MERGE_PROC`, `OBJFETCH_PROC`, `UPDATE_PROC`, `DELETE_PROC`, `CONNECTBY_PROC`, `DO_PROC`, `BUILD_SCHEMA_PROC`, `selected_upd_list`, `XASL_MULTI_UPDATE_AGG`, CTE recursive part, `bptr_list/fptr_list` present | NO_PARALLEL | `:638-699` |

### New surface (no existing wiki reference)

These files/symbols have no current wiki page. Listed for future dedicated ingest:

- `parallel_scan::input_handler_index` — leaf-mutex cursor design
- `parallel_scan::input_handler_list` — sector pre-split + membuf fallback
- `parallel_scan::slot_iterator_index` — 1026 LOC, biggest new file; key iteration + heap-lookup handshake
- `parallel_scan::slot_iterator_list` — tuple iteration within sector range
- `parallel_scan::SCAN_TYPE` enum + `scan_traits<ST>` template framework
- `PRM_ID_PARALLEL_SCAN_PAGE_THRESHOLD` system parameter (no dedicated page for system params exists yet — candidate for `components/system-parameters.md`)

## Review discussion highlights

All substantive review threads are between the author (`@xmilex-git`) and internal reviewers (`@hornetmj`, `@shparkcubrid`, `@youngjinj`). The external `pr_7062_code_review.md` document captures the full design rationale. From the inline threads (37 comments), the recurring themes:

- **Mutex ordering for index-scan input handler.** A reviewer raised concern about "mutex held while acquiring page latch" (`px_scan_input_handler_index.cpp:214`). Author clarified that the existing `parallel_heap_scan::input_handler_single_table` already used the same `std::unique_lock(m_vpid_mutex)` + `heap_page_next_fix_old` pattern and that **CUBRID has no explicit page-latch → mutex ordering rule**; the new index handler follows precedent. This is a latent convention worth documenting but was accepted as-is for this PR.

- **`thread_local static` class-member pattern for list handler.** Reviewer asked whether the `thread_local static ftab_set *m_tl_ftab_set` pattern was new. Author: introduced originally in the parallel heap-scan work committed 2026-03-27 (`45730b900`, i.e. [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911]]); list handler reuses the same convention. Parallel scan is at most one-per-thread; subsequent scans in NL-join contexts run as `scan_ptr`, so TLS singletons are safe.

- **`btree_read_fixed_portion_of_non_leaf_record` is file-static.** Reviewer suggested using it for non-leaf parsing in `input_handler_index`. Author: it's `static` in `src/storage/btree.c` with no public wrapper in `btree.h`. Exposing it is a separate storage-module task. The current PR parses non-leaf records directly via `spage_get_record` + `OR_GET_INT` / `OR_GET_SHORT` until a storage-side refactor lands.

- **Null-deref fix in RESULT_TYPE fallback.** A reviewer caught that when `XASL_SNAPSHOT × LIST` was nominally possible (before the checker contract existed), the `else` branch set `S_PARALLEL_LIST_SCAN` with `manager == nullptr`, leading to potential crash in release builds. Author agreed and fixed in `195155bc5` — moved to the pattern used in `scan_open_parallel_index_scan` where type/manager/trace_storage are set only after `manager::open()` succeeds.

- **Same fix pattern applied to INDEX path.** A different reviewer found the same class of bug in the index path; fixed in `8799d77` by releasing reserved workers + returning `NO_ERROR` before touching `scan_id` when neither `MERGEABLE_LIST` nor `BUILDVALUE_OPT` flag is set.

- **One STX-allocated pointer that must not be freed.** Author repeated clarification (three separate threads): a particular structure pointer is allocated during `stream → xasl` unpacking and must not be freed at the end of parallel-scan teardown. Reviewer accepted.

- **`open()` is called at most once.** Clarified on `px_scan.cpp:1786`; no lifecycle race.

- **`XASL_ANALYTIC_SKIP_SORT` affects index scan only.** Clarified on `px_scan_checker.cpp:915` — the flag's relationship to sort-order-elision only matters for the cursor-ordered paths.

The design-convergence highlights worth calling out (from review doc §7): `compute_parallel_degree` has a hard floor at 2 workers (falls back to single-thread below that), filtered-index detection goes through schema_manager rather than parse-tree inspection, `MERGELIST_PROC` propagation handles the full subtree, `need_count_only` aggregate path blocks parallel index scan, and NULL `midxkey` is rejected outright.

## Reconciliation Plan

**Status: WRITTEN, NOT APPLIED.** Execute via "apply reconciliation for PR #7062" once the PR merges (or earlier if the user opts in). Each item below is self-contained — executable without re-reading the PR.

### Existing component pages — to be UPDATED on merge

#### [[components/parallel-heap-scan]] — rename + repurpose

- **Current state:** Top-level page for the heap-specific parallel scan, frontmatter `path: "src/query/parallel/px_heap_scan/"`.
- **Proposed change:** Rename file to `wiki/components/parallel-scan.md` (git mv) + rewrite as the generalised parent page. Retain a short "## Heap-specific details" section. Update frontmatter `path` to `src/query/parallel/px_scan/`. Update `key_files` block to `px_scan.hpp`, `px_scan.cpp`, `px_scan_type.hpp`, etc.
- **RESULT_TYPE enum update:** the code block `COUNT_DISTINCT = 0x3, // (fast) for UPDATE STATISTICS — aggregate-only` becomes `BUILDVALUE_OPT = 0x3, // (fast) aggregate-only (UPDATE STATISTICS, COUNT DISTINCT)`.
- **Result Mode Selection block (lines 77-80):** replace `ACCESS_SPEC_FLAG_COUNT_DISTINCT` with `ACCESS_SPEC_FLAG_BUILDVALUE_OPT`.
- **Manager Template Fields table:** change `manager<RESULT_TYPE>` to `manager<RESULT_TYPE, SCAN_TYPE>`.
- **Callout type:** `[!update]` citing PR #7062 + merge sha (once known).

#### [[components/parallel-heap-scan-input-handler]] — rename + expand

- **Current state:** Covers only `input_handler_ftabs` (heap variant).
- **Proposed change:** Rename file to `parallel-scan-input-handler.md`; section-split into "Heap variant (`input_handler_heap`, renamed from `input_handler_ftabs`)", "List variant (`input_handler_list`)", "Index variant (`input_handler_index`)". Per-variant summary with the design decisions from `pr_7062_code_review.md` §2.1-2.3.
- **Callout type:** `[!update]`.

#### [[components/parallel-heap-scan-slot-iterator]] — rename + expand

- Rename to `parallel-scan-slot-iterator.md`; add "List variant" (200 LOC) and "Index variant" (1026 LOC) subsections.
- **Callout type:** `[!update]`.

#### [[components/parallel-heap-scan-task]] — rename + template signature

- Rename to `parallel-scan-task.md`. Update `task<RESULT_TYPE>` → `task<RESULT_TYPE, SCAN_TYPE>`.
- **Callout type:** `[!update]`.

#### [[components/parallel-heap-scan-result-handler]] — rename + enum rename

- Rename to `parallel-scan-result-handler.md`. Every `COUNT_DISTINCT` → `BUILDVALUE_OPT`. Add `SCAN_TYPE` second template parameter. Document the 7-of-9 valid matrix.
- **Callout type:** `[!update]`.

#### [[components/parallel-heap-scan-join-info]] — rename only

- Rename to `parallel-scan-join-info.md`. Path update, nothing else.
- **Callout type:** `[!update]`.

#### [[components/parallel-heap-scan-support]] — rename + checker expansion

- Rename to `parallel-scan-support.md`. Checker section gets the index-specific and XASL-level blocker tables from this PR page's "Checker" section.
- **Callout type:** `[!update]`.

#### [[components/scan-manager]]

- Add `S_PARALLEL_LIST_SCAN` and `S_PARALLEL_INDEX_SCAN` rows to the Scan Types table (after existing `S_PARALLEL_HEAP_SCAN`).
- Add `pllsid_parallel: PARALLEL_LIST_SCAN_ID` and `pisid: PARALLEL_INDEX_SCAN_ID` rows to the `SCAN_ID` union code block.
- Update the `scan_next_scan` dispatch summary to include the two new cases.
- **Callout type:** `[!update]`.

#### [[components/xasl]]

- `ACCESS_SPEC_FLAG_*` table (added as an incidental enhancement in this ingest — see below): update on merge so `NO_PARALLEL_HEAP_SCAN` → `NO_PARALLEL_SCAN` and `COUNT_DISTINCT` → `BUILDVALUE_OPT`.
- **Callout type:** `[!contradiction]` for the two renames (baseline → post-merge).

#### [[components/parallel-query-checker]]

- Expand to cover the new checker rules tables (index-specific + XASL-tree-level) from this PR page. These are the key reference for "why doesn't my query parallelise".
- **Callout type:** `[!update]`.

#### [[components/parallel-query]]

- Overview: change "parallel heap scan" to "parallel scan (heap / list / index)" wherever it reads as the exclusive capability.
- **Callout type:** `[!update]`.

#### [[components/parallel-query-executor]]

- Dispatch summary: add `S_PARALLEL_LIST_SCAN` and `S_PARALLEL_INDEX_SCAN` cases.
- **Callout type:** `[!update]`.

#### [[components/btree]]

- Add a "Parallel index scan hooks" section covering what was added to `btree.c/.h` for leaf-chain cursor management. Cite the "vertical traversal inlined" contract (format of non-leaf records is now part of the parallel-scan contract).
- **Callout type:** `[!update]`.

#### [[components/parser]] and [[components/xasl-generation]]

- Note the `NO_PARALLEL_HEAP_SCAN` → `NO_PARALLEL_SCAN` rename across parse-tree / XASL generation.
- **Callout type:** `[!contradiction]`.

#### [[components/query-dump]]

- Note the per-worker-per-partition trace output now distinguishes heap/list/index separately.
- **Callout type:** `[!update]`.

#### [[components/_index]]

- Replace `parallel-heap-scan*` entries with `parallel-scan*` entries (both in listing and nav).
- **Callout type:** `[!update]`.

### Existing source pages — to be UPDATED on merge

#### [[sources/cubrid-src-query-parallel]]

- Rewrite the `px_heap_scan/` section header to `px_scan/`. Add paragraphs for the three new input handlers and two new slot iterators. Add the new system parameter. Note the two deleted files (1347 LOC removed).
- **Callout type:** `[!update]`.

#### [[sources/cubrid-src-query]] and [[sources/cubrid-src-parser]] and [[sources/cubrid-src-storage]]

- Note the integration changes (`scan_manager.c/h`, `query_executor.c`, parser hint rename, `btree.c/h` additions).
- **Callout type:** `[!update]`.

### New component pages to CREATE on merge

- `wiki/components/parallel-scan-input-handler-index.md`
- `wiki/components/parallel-scan-input-handler-list.md`
- `wiki/components/parallel-scan-slot-iterator-index.md` (1026 LOC file — deserves its own page)
- `wiki/components/parallel-scan-slot-iterator-list.md`
- `wiki/components/parallel-scan-type.md` (cover `SCAN_TYPE` enum + `scan_traits<ST>` template framework)

### Candidate ADR

`wiki/decisions/NNNN-parallel-scan-type-extension.md` — the `manager<RT, ST>` template generalisation, the SCAN_ID union retention over unification, and the `XASL_SNAPSHOT × LIST/INDEX` checker carve-out are all clean ADR material. Judgment call.

### Baseline bump plan

- Before: `175442fc858bd0075165729756745be6f8928036`
- After (on merge): `<PR-7062 merge commit>` — not yet known; fill on execution
- Log entry: `## [YYYY-MM-DD] baseline-bump | 175442fc → <new-sha7> (PR #7062)`

## Pages Reconciled

**None yet — PR is open.** This section will be populated when the Reconciliation Plan is executed, copying the per-page entries above with the `applied_commit` annotations.

## Incidental wiki enhancements

**1 enhancement applied** (baseline truths surfaced during deep code analysis, independent of this PR's changes):

- **[[components/xasl]]** — added a complete `ACCESS_SPEC_FLAG_*` table (all 7 flags: `NONE`, `FOR_UPDATE`, `NO_PARALLEL_HEAP_SCAN`, `NUM_PARALLEL_THREADS`, `MERGEABLE_LIST`, `COUNT_DISTINCT`, `ONLY_MIN_MAX_SCAN`, `FORCE_FIXED_SCAN`) with bit values and semantics. Previously the page only referenced two by name in prose. Also added a paragraph explaining the `PT_HINT_* → PT_SPEC_FLAG_* → ACCESS_SPEC_FLAG_*` propagation pipeline between parser and XASL generation. These are baseline facts reflecting the current state at commit `175442fc8`; on PR #7062 merge, the table will need the two renames (`NO_PARALLEL_HEAP_SCAN → NO_PARALLEL_SCAN`, `COUNT_DISTINCT → BUILDVALUE_OPT`) applied via Reconciliation Plan entry above.

Additional candidate enhancements (not applied this round — bounded scope):
- The `MRO / ISS / ILS` index-scan optimizations (referenced by the checker) have no dedicated wiki coverage and are acronyms scattered across `components/btree.md` and `components/scan-manager.md`.
- The "no page-latch → mutex ordering rule" CUBRID convention (surfaced in review comments) is a latent cross-cutting concern not documented.
- The `thread_local static` ftab_set pattern (from PR #6911) is a convention worth its own short page.

These are left for future ingests to pick up.

## Baseline impact

- Before: `175442fc858bd0075165729756745be6f8928036`
- After:  `175442fc858bd0075165729756745be6f8928036` (unchanged — PR not merged)
- Bump triggered: **false** (PR is open)
- Logged: no baseline-bump entry — none occurred

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/7062
- Jira: CBRD-26722
- Prior related PR: [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911]] (same author, same subsystem — I/O bottleneck removal via sector pre-split, the foundation that this PR generalises)
- External design doc: https://github.com/user-attachments/files/26920618/pr_7062_code_review.md
- Primary affected components: [[components/parallel-heap-scan]], [[components/scan-manager]], [[components/parallel-query-checker]], [[components/xasl]]
- Primary affected sources: [[sources/cubrid-src-query-parallel]]
