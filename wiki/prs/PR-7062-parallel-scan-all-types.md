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
base_sha: "0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0"
head_sha: "58fab454f9ed8fe14bfe570f44d19be18ee5e036"
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
incidental_enhancements_count: 4
last_reingested: 2026-05-07
last_reingested_head: "58fab454f9ed8fe14bfe570f44d19be18ee5e036"
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
updated: 2026-05-07
status: open
---

# PR #7062 — Expand parallel heap scan to parallel scan (index, heap, list) (CBRD-26722)

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `OPEN` (non-draft) · **Author:** `@xmilex-git` · **Jira:** CBRD-26722
> **Base → Head:** `develop` (`0be6cdf6`) → `parallel_scan_all` (`7fdb82099`)
> **Stats:** 52 files changed, **91 commits**, ≈ **+6,061 / −1,753 LOC** (HEAD `7fdb82099`; 4 commits beyond the `0f8a107bb` review snapshot covering the CBRD-26722 parallel-index-on-partitioned-tables fix)

> [!update] Re-ingest 2026-04-29 (HEAD `0f8a107bb`)
> External code-review document was rewritten to reflect the live branch HEAD `0f8a107bb` (22 commits beyond the original `c28c5945a` snapshot). This page is updated in place to match: stats refreshed, frontmatter `head_sha` / `base_sha` corrected, BUILDVALUE_OPT aggregate count fixed (13, not 11), system-parameter section re-confirmed as TWO-parameter split, post-`c28c5945a` commit summary added (greptile P1 fixes, fallback patterns, latch-couple, double-fix removal, lazy CAS membuf claim, `qfile_collect_list_sector_info` reuse). No baseline bump (PR still OPEN). Reconciliation Plan unchanged in shape — still pending merge.

> [!update] Pending-buffer reconciliation 2026-05-07 (HEAD `58fab454f`)
> Three pending divergences from `~/dev/cubrid/.claude/wiki-updates/pending.md` applied to this page: (1) leaf-cursor model rewritten as **per-range vertical descent + drain CV** (the prior shared-cursor description still applies intra-range, the multi-range path no longer relies on `next_vpid` chain — see Behavioral § "Index scan — per-range vertical descent + drain CV"); (2) range conversion retitled to `input_handler_index::convert_all_key_ranges` with three-step pipeline (truncation collapse → DESC swap → storage-order sort) — Deep analysis § "Range cursor optimization"; (3) trace counter parity note added — `key_qualified_rows`/`read_rows` count visible OIDs, not slot keys, with both serial and parallel increment sites cited. New HPP fields backing per-range descent: `m_active_workers`, `m_pending_advance_idx`, `m_advance_in_progress`, `m_advance_cv`. PR still OPEN — no PR-reconciliation, no baseline bump; `head_sha` advanced from `7fdb82099` → `58fab454f` to track 4 follow-on CBRD-26722 commits (`7fdb82099` → `05d091c66` → `f74891494` → `fc1b51091` → `d117dd946` → `58fab454f`).

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

- **Index scan — per-range vertical descent + drain CV (post `58fab454f`).** The original "shared leaf cursor `VPID m_current_leaf_vpid` protected by `std::mutex m_leaf_mutex`" model still applies *within a single range* (`R_KEY` / single-range `R_RANGE`), but multi-range queries (`R_KEYLIST` / `R_RANGELIST`) now transition between ranges via a **fresh root→leaf descent per `range_idx`**, gated by a drain barrier instead of the leaf chain. `input_handler_index::descend_to_first_leaf(thread_p, worker_scan_id, range_idx, out_leaf)` (`px_scan_input_handler_index.cpp:256-409`) runs root→leaf descent for the requested range; `release_leaf_and_maybe_advance(thread_p, worker_scan_id, local_advance_target)` (`:504-548`) decrements `m_active_workers`, and the next-range driver waits on `m_advance_cv` until `m_active_workers == 0` before issuing the new descent. New HPP fields backing the model (`px_scan_input_handler_index.hpp:127-130`): `int m_active_workers`, `int m_pending_advance_idx`, `bool m_advance_in_progress`, `std::condition_variable m_advance_cv`. `m_leaf_mutex` is retained — it still guards `m_current_leaf_vpid` republish during a single-range walk — but it no longer crosses range boundaries.
  > [!update] Superseded design note (commit `58fab454f`)
  > The earlier sentence "the key-range-partition alternative was considered and rejected (skew sensitivity + setup cost of bounding-key lookups)" is **superseded**: per-range descent was de-facto adopted for `R_KEYLIST` / `R_RANGELIST`. The rejection still holds for *intra-range* partitioning by interior key bounds; what changed is that ranges themselves are now boundary-walked individually rather than concatenated through the leaf chain. The mutex still amortises to noise on large indexes; small indexes still hit the `parallel_scan_page_threshold` gate.

- **Index scan — vertical traversal is inlined, not via `btree_find_first()`.** Root-to-start-leaf descent happens on main thread in `init_on_main` (`px_scan_input_handler_index.cpp:51-173`) via `spage_get_record` + `OR_GET_INT` / `OR_GET_SHORT` raw parsing of non-leaf records. B+tree internal record format becomes a **synchronisation-sensitive contract** — any format change in storage will force an edit in the parallel-scan input handler.

- **List scan — temp-file sectors pre-split.** On open, `file_get_all_data_sectors` collects the disk-resident sectors of the backing `QFILE_LIST_ID` temp file, and the `ftab_set` is split evenly across workers. Workers then iterate their owned sector range without cross-thread coordination. **Membuf fallback:** if the list is small enough to live entirely in memory (`m_has_membuf == true`, no `temp_vfid`), sector partition is impossible and the scan falls back to single-threaded. **Dependent lists:** when the list references another (`dependent_list_id`), both temp files are merged into one collector so no sectors leak.

- **Two system parameters, not one** (correction from first-pass reading). Final state after merge:
  - `PRM_ID_PARALLEL_SCAN_PAGE_THRESHOLD` (name: `parallel_scan_page_threshold`): flags `PRM_FOR_SERVER | PRM_HIDDEN`, default `2048`, min `2048`, max `INT_MAX`. Hidden server-side kill-switch; used by `scan_manager` / `compute_parallel_degree`.
  - `PRM_ID_PARALLEL_INDEX_SCAN_PAGE_THRESHOLD` (name: `parallel_index_scan_page_threshold`): flags `PRM_FOR_CLIENT | PRM_USER_CHANGE | PRM_FOR_SESSION`, default `32`, min `32`, max `INT_MAX`. **Separate client-side tuning knob** consumed only by the optimizer cost model in `qo_apply_parallel_index_scan_threshold` (`plan_generation.c`). End-users can set this per session to influence index-scan cost decisions without touching the server param.
  - `PRM_ID_PARALLELISM` flags changed: was `PRM_FOR_SERVER` only, now `PRM_FOR_SERVER | PRM_FOR_CLIENT` — clients can now set the parallelism hint.
  - History: originally introduced as a single `parallel_index_scan_page_threshold` default `32` (commit `8827dd968`); the server-side default was raised to `2048` after measurement showed leaf-mutex contention outweighs parallelism on small indexes (`e17db63eb`); finally split so the optimizer's input and the runtime kill-switch can tune independently (`9551c8356`).

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

## Deep analysis — supplementary findings (post initial ingest)

This section captures findings from a full line-by-line read of the 10,396-line diff across 5 parallel sub-agent passes. The content above was written from the external review doc; items below are corrections and additions only visible from the code itself.

### Corrections to the initial write-up

- **BUILDVALUE_OPT scope is far wider than "COUNT DISTINCT"** (the previous claim). The rename from `COUNT_DISTINCT` → `BUILDVALUE_OPT` reflects an actual capability expansion. The new `is_buildvalue_opt_supported_function()` checker whitelists **13 aggregate functions**: `PT_COUNT_STAR`, `PT_COUNT`, `PT_MIN`, `PT_MAX`, `PT_SUM`, `PT_AVG`, `PT_STDDEV`, `PT_STDDEV_POP`, `PT_STDDEV_SAMP`, `PT_VARIANCE`, `PT_VAR_POP`, `PT_VAR_SAMP`, `PT_GROUP_CONCAT`. DISTINCT aggregates (non-MIN/MAX) use per-worker `qfile` sort-distinct lists; MIN/MAX use collation-aware compare + coerce; SUM/AVG/STDDEV/VARIANCE use `qdata_add_dbval` with per-row domain-resolution fallback. DISTINCT loop optimisation (commit `c047445c8`, greptile review feedback) reuses the first-operand fetch instead of refetching per row. The previous one-liner in this page understated the change significantly.
- **Two system parameters, not one** — corrected in the Behavioral section above.
- **`qexec_resolve_domains_for_aggregation_for_parallel_heap_scan_buildvalue_proc` retains the `heap_scan` substring in its name post-merge** — the function is called from the new `BUILDVALUE_OPT` `write()` path (`px_scan_result_handler.cpp:6425`) but its identifier was never renamed. Latent inconsistency; cleanup candidate for a follow-up PR.

### Lifecycle / resource-ownership invariants not in the review doc

- **Placement-new + explicit destructor + db_private_free triplet.** `manager`, `input_handler`, and `result_handler` are allocated via `db_private_alloc()`, constructed via placement-new, destroyed via an explicit `~manager()` call, then freed via `db_private_free`. Error paths in `scan_open_parallel_*` must run the destructor before the free or worker-pool reservations leak. `px_scan.cpp:~2550-2570` (heap), `~3100-3150` (index/list). Destructor itself must call `worker_manager->release_workers()` — not optional.
- **Deferred promotion for index scan** is distinct from heap/list. `scan_open_parallel_index_scan` allocates a `parallel_index_scan_pending` struct, stashes it in `scan_id->s.isid.parallel_pending`, and **does not** open the manager. `scan_start_scan` later calls `scan_try_promote_parallel_index_scan` which consumes the pending struct to attempt worker reservation. If the reservation fails (workers unavailable, `need_count_only` flipped since prepare, etc.), the `isid/pisid` union fallback is clean. Pending state is freed by `scan_close_parallel_index_scan` if never consumed. This pattern exists because aggregate fast-path decisions (especially `need_count_only`) may be made between `open` and `start` and should be respected. Heap and list have no analogous pending state.
- **`scan_clear_parallel_index_pending` count(\*)-safety hook.** `qexec_intprt_fnc` invokes this helper when `need_count_only == true` to discard any stashed parallel-index promotion — forces the single-thread `count(*)` fast path. Without this the pending struct would attempt promotion mid-execution.
- **Trace storage lazy allocation.** `trace_storage` is allocated not at `scan_open()` but on the first `scan_reset_scan_block_parallel_*` call when `thread_p->on_trace == true`. If a scan errors before reset, trace stats are lost — no fallback to earlier allocation.
- **Error propagation MOVES rather than copies.** Worker errors reach main via `m_err_messages_p->move_top_error_message_to_this()`. Only the first worker to acquire the lock has its error recorded; subsequent worker errors are silently discarded. No error queue.
- **`scan_regu_key_to_index_key` publicity change.** Was `static` in `scan_manager.c`; now `extern` and declared in `scan_manager.h`. Required by the parallel-index input handler for key-range setup.

### Index slot iterator — invariants beyond §2.2 of review doc

- **Fence key skip is required for correctness, not just performance.** Every leaf has sentinel fence records at boundaries (detected via `btree_leaf_record_is_fence`). These duplicate edge entries across adjacent leaves; skipping them prevents double-counting in GROUP BY / aggregates under parallel iteration. Fence check happens only **after** `spage_get_record` succeeds (`px_scan_slot_iterator_index.cpp:7969, 7979`).
- **Multi-OID per key.** Non-unique indexes may store multiple OIDs per leaf key (overflow chain). `btree_key_process_objects()` callback (`collect_oid_callback` at 7757-7776) collects visible OIDs through MVCC snapshot filtering, then each OID is processed by `process_oid()` — heap fetch, predicate eval, val_list fill.
- **Covering index vs heap-fetch handshake.** Covering-index path reads output directly from the B-tree key via `btree_attrinfo_read_dbvalues` (7811). Non-covering path does `heap_get_visible_version` (7856). Two different filter functions: `eval_key_filter` for covering, `eval_data_filter` for non-covering (7797-7846 vs 8082-8118).
- **Range cursor optimization.** When multiple ranges are specified, keys arrive in sorted B-tree order, so `m_current_range_idx` advances monotonically — once a range's upper bound is passed it is never revisited. Range conversion + sort is now `input_handler_index::convert_all_key_ranges(thread_p, worker_scan_id)` at `px_scan_input_handler_index.cpp:71-247` (post `58fab454f`) — owned by `input_handler`, no longer by `slot_iterator`. The function is a three-step pipeline: (1) **prefix-truncation collapse** (`GT_*` → `GE_*`, `LT_*` → `LE_*`, lines 134-153) for prefix indexes whose bounds were truncated; (2) **`part_key_desc` swap** (lines 179-192) — `range_reverse(range)` + `swap(key1, key2)` mirroring `btree_prepare_bts`; (3) **storage-order sort** (lines 196-245) — bubble-sort by `key1` under `btree_compare_key` so leaf-chain forward traversal walks ranges in cursor-friendly order. `slot_iterator_index` no longer owns `m_key_val_ranges`; it reads via `m_input_handler->get_key_val_ranges()` / `get_num_key_ranges()` / `is_part_key_desc()`.
- **`part_key_desc` bound swap.** If the last partial-key domain is DESC, the B-tree physically stores keys in reverse order for that domain. The iterator swaps lower/upper bounds and inverts range operators (GT↔LT) exactly like `btree_prepare_bts` (7449-7462). Correctness critical; not obvious from SQL semantics.
- **NULL midxkey rejection with Oracle-empty-string exception.** Replicates `btree_apply_key_range_and_filter`: NULL in the `num_index_term`-th composite-key element fails `<, <=, >, >=, BETWEEN`. BUT: if `PRM_ID_ORACLE_STYLE_EMPTY_STRING` is set and the element is an empty-string char/bit, it's treated as non-NULL (8041-8077).
- **Reversal semantics for `use_desc_index` (checker-blocked, code-ready).** The iterator physically supports descending traversal (slot counts down from `m_num_keys`, leaf chain reads `prev_vpid`). The checker currently blocks `use_desc_index` from parallel outright (precautionary; review-doc §3.1 says "relatively untested path"). Enabling later requires no iterator change.
- **No explicit handling for mid-iteration btree splits/merges.** If a leaf splits after a worker reads `next_vpid` but before the next worker fixes it, the new sibling is silently skipped. Acceptable because (a) filtering is stateless and (b) B-tree pages are not reused within a session — the stale page will simply appear empty or missing, not corrupt. **Caveat**: relies on the "pages not reused" invariant holding.

### Input handlers — invariants beyond §2.3 of review doc

- **List handler membuf-worker assignment.** Worker 0 always becomes the membuf-worker (`m_tl_is_membuf_worker = (idx == 0 && m_has_membuf)`). Membuf pages (volid `NULL_VOLID`) are consumed by worker 0 in Phase 1 before it joins the disk-sector iteration in Phase 2. Other workers skip Phase 1 entirely.
- **List handler dependent-list chain.** When a `QFILE_LIST_ID` has `dependent_list_id` set (e.g., join intermediate feeding another intermediate), the handler walks the chain via `for (dep = list_id->dependent_list_id; dep; dep = dep->dependent_list_id)` and appends every dependent's sectors to the shared collector via the new `ftab_set::append_from_collector()` method. **Risk**: if a dependent is still being written, sector collection can be incomplete. Mitigation: executor guarantees full materialisation before parallel-list open.
- **Overflow-tuple page skipping.** After computing a page VPID from the sector bitmap, the handler briefly fixes the page and checks `QFILE_GET_TUPLE_COUNT`. If the marker is `QFILE_OVERFLOW_TUPLE_COUNT_FLAG == -2`, the page is skipped (`continue`) — overflow pages are already consumed by the primary page's `qfile_get_tuple` handler, so duplicate processing would violate correctness. One extra read-only I/O per overflow page, unavoidable given sector-level partitioning has no tuple awareness.
- **Heap handler — pure rename with one addition.** The renamed `px_scan_input_handler_heap.cpp` is byte-for-byte logic-identical to baseline `px_heap_scan_input_handler_ftabs.cpp` — EXCEPT the new `ftab_set::append_from_collector()` method added to the shared `ftab_set` class. Heap itself never calls it; exists solely to support the list-handler dependent-chain case.
- **Atomic index assignment is seq_cst.** `m_splited_ftab_set_idx.fetch_add(1)` uses default `std::memory_order_seq_cst`. Called once per worker at init; not perf-critical; conservative ordering is fine.
- **Lock order is mutex → page latch.** The index handler's `m_leaf_mutex` is held during `pgbuf_fix` of the leaf page. CUBRID has no explicit rule for mutex/latch ordering; this precedent was set by the heap `input_handler_single_table` in [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911]] and carried forward. Follow-up question in review comments (documented above) accepted the precedent.

### BTree integration — new public helpers

- **`btree_leaf_record_is_fence`** (`btree.c:1911-1927`, declared in `btree.h:831`): now public. The fence-record detector needed by parallel-scan slot iteration.
- **`btree_key_process_objects` + `BTREE_PROCESS_OBJECT_FUNCTION` typedef** (`btree.c:520-537, 24503`, declared in `btree.h:835-844`): moved from file-static to public API. Function changed from `static int` to `int`. Needed by `collect_oid_callback` in the index slot iterator.

### Optimizer cost logic — new 137-line function

`qo_apply_parallel_index_scan_threshold` (`plan_generation.c:2925-3061`) is **not covered by the review doc**:

- Descends the plan tree via `qo_find_driving_scan_plan` to locate the leftmost `SCAN_PLANTYPE_SCAN`.
- Computes selectivity via `QO_TERM_SELECTIVITY` product over the driving table's filter predicates.
- Cost formula: `metric = ceil(sel * index_leaf_pages)`. If `metric < threshold`, marks the spec `NO_PARALLEL_SCAN`.
- Degree formula: `floor(log2(metric / threshold)) + 2`, capped by `PRM_ID_PARALLELISM` / `PRM_MAX_PARALLELISM`. Matches the runtime `compute_parallel_degree` formula.
- Respects `PARALLEL(N)` hint: if `N <= 1`, mark `NO_PARALLEL`.
- Added `#include <math.h>` for `ceil()`.

### Trace handler — new 434-line file

- Per-worker `child_stats` struct with heap fields (`fetches`, `ioreads`, `fetch_time`, `read_rows`, `qualified_rows`, `elapsed_time`) plus index-specific fields (`read_keys`, `qualified_keys`, `key_qualified_rows`, `data_qualified_rows`, `elapsed_lookup`, `covered_index`, `multi_range_opt`, `index_skip_scan`, `loose_index_scan`, `need_count_only`).
  > [!update] Trace counter parity — granularity bug fixed (commit `58fab454f`)
  > **Unit:** `key_qualified_rows` and `read_rows` count **visible OIDs**, not slot keys. The serial path increments per visible OID at `scan_manager.c:6279`; the parallel path was previously incrementing per leaf-key slot at `px_scan_slot_iterator_index.cpp:691,696` — same trace label, different units. KEYLIST/RANGELIST + lookup queries (e.g. dup-key `IN` over 5 keys × ~10 000 OIDs each) reported `rows: 1..2` per worker vs `rows: 50000` serial. Fix: removed slot-level `++`, added `m_scan_id->scan_stats.key_qualified_rows += m_slot_oids.size()` and `m_scan_id->scan_stats.read_rows += m_slot_oids.size()` after the `btree_key_process_objects` collector populates `m_slot_oids` (`px_scan_slot_iterator_index.cpp:731-732`, post-empty-skip). Final result counts were always correct; only the trace counter was wrong. Cite both increment sites — `scan_manager.c:6279` (serial) and `px_scan_slot_iterator_index.cpp:731-732` (parallel) — when reviewing future trace changes so unit drift is caught.
- `merge_stats` sums into main via `perfmon_add_at_offset_to_local` for perfmon integration. Boolean flags OR'd (covered_index etc. set if ANY worker detected).
- `accumulative_trace_storage` retains per-worker history across scan iterations.
- `dump_stats_json` emits 70-line JSON object with `parallel_workers` count, time ranges, key/row ranges, gather mode (`mergeable list` / `row by row` / `buildvalue` / `unknown`), per-index flags — visible in EXPLAIN JSON output.
- `dump_stats_text` produces text-mode equivalent.
- No Jansson in the .cpp file; uses `json_pack` / `json_object_set_new` (Jansson already via shared perfmon path).

### Code concerns / smells spotted

- **Dangling `heap_scan` substring in a public function name.** `qexec_resolve_domains_for_aggregation_for_parallel_heap_scan_buildvalue_proc` (see Corrections). Not a bug but an inconsistency with the rest of the rename.
- **Old TODO survives the PR unchanged.** `src/base/system_parameter.c:~250` retains `sprintf(newval, "%d", 32); /* TODO: 0 ? */` despite the parameter being renamed/split. No resolution in PR history.
- **Redundant field clears in `scan_reset_scan_block_parallel_heap_scan`.** Comments in the diff mark several `single_fetched`/`null_fetched`/`qualified_block` assignments as "TODO: cleared by scan_reset_scan_block; redundant?". Dead-code or duplicated effort — neither confirmed nor removed in this PR.
- **Index-handler fallback label uses generic `ER_FAILED` + `er_clear()`.** The `init_on_main` fallback path sets a generic error then clears the stack, relying on callers not checking `er_errid()` afterwards. Fragile — if any caller pattern inverts, the failure becomes silent.
- **`write_initialize` on DISTINCT aggregates can return without setting an interrupt flag in some branches.** Observed in earlier commits; later fixed in the same PR series, but the window of undefined behavior in the intermediate state is worth flagging for any bisect.
- **No assert that the fixed page is a leaf.** Index iterator's `pgbuf_fix(m_current_leaf_vpid)` does not validate `node_level == 1` after fix. Corruption (pointing at a non-leaf) would lead to wrong data being read via `spage_get_record` rather than a hard failure. `assert(btree_get_node_header(...)->node_level == 1)` after fix would harden the contract.
- **`set_page()` defers key range conversion to first call.** If an empty-index scan never calls `set_page`, ranges are never converted. Safe in practice but creates an implicit ordering dependency.
- **No NULL guard on `indx_info->key_info.key_ranges`** when `key_cnt > 0`. Mirrors the non-parallel path's assumption, but worth auditing upstream INDX_INFO construction to confirm the invariant holds.
- **Non-leaf record byte layout is a hidden contract.** The `OR_GET_INT(rec.data)` + `OR_GET_SHORT(rec.data + OR_INT_SIZE)` pattern in `init_on_main` (parsing pageid + volid directly from the record buffer) means any future change to the B-tree non-leaf record format silently breaks this reader. No abstraction layer protects it.

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

**4 enhancements applied** (baseline truths surfaced during deep code analysis, independent of this PR's changes). Each is a fact about the tree at the current baseline `175442fc8` — the PR's changes will retroactively require small updates to two of them (flagged in the Reconciliation Plan).

1. **[[components/xasl]]** — added a complete `ACCESS_SPEC_FLAG_*` table (all 7 flags: `NONE`, `FOR_UPDATE`, `NO_PARALLEL_HEAP_SCAN`, `NUM_PARALLEL_THREADS`, `MERGEABLE_LIST`, `COUNT_DISTINCT`, `ONLY_MIN_MAX_SCAN`, `FORCE_FIXED_SCAN`) with bit values and semantics. Previously the page only referenced two by name in prose. Also added a paragraph explaining the `PT_HINT_* → PT_SPEC_FLAG_* → ACCESS_SPEC_FLAG_*` propagation pipeline between parser and XASL generation.

2. **[[components/btree]]** — expanded the "Fence Keys" section with the `btree_leaf_record_is_fence` public API contract and the double-counting hazard that motivates it; added new "Leaf-page header and leaf-chain pointers" section documenting `BTREE_NODE_HEADER.next_vpid/prev_vpid/node_level/num_keys` (what makes the leaf chain traversable); added new "Non-leaf record byte layout" section with the 6-byte `(pageid: INT32, volid: INT16)` prefix contract that direct parsers (e.g. parallel scan's root-to-leaf descent) depend on; added new "MVCC visibility filtering during B-tree iteration" section documenting `btree_mvcc_info_to_heap_mvcc_header` and `btree_key_process_objects` with `BTREE_PROCESS_OBJECT_FUNCTION` callback typedef.

3. **[[components/list-file]]** — expanded the `QFILE_LIST_ID` section with three new subsections: "Dependent-list chain" (the `dependent_list_id` linked-list pattern used by nested-loop join intermediates), "Membuf vs disk backing" (the two-backend storage model of `QMGR_TEMP_FILE` and the membuf-first-then-spill transition), and "`QFILE_OVERFLOW_TUPLE_COUNT_FLAG`" (the `-2` sentinel marking overflow pages that must be skipped during sector-level iteration).

4. **[[components/file-manager]]** — added new "Data-sector harvesting (`file_get_all_data_sectors`)" subsection under Layer 1 documenting the `FILE_FTAB_COLLECTOR` API, the PART_FTAB+FULL_FTAB walk that produces the per-sector page bitmaps, the `VFID *` argument type (widened from `HFID *` during review for non-heap file reusability), and the main-thread / single-threaded-at-open contract.

### Additional candidate enhancements (not applied this round)

- `MRO / ISS / ILS` index-scan optimizations — referenced by the checker in this PR but have no dedicated wiki coverage. Scattered across `components/btree.md` and `components/scan-manager.md` as acronyms only.
- The "no page-latch → mutex ordering rule" CUBRID convention — surfaced in review comments as a latent cross-cutting convention not documented anywhere.
- The `thread_local static` class-member pattern for parallel-scan workers (originating in [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911]]) — convention worth a short dedicated page.
- Deferred-promotion pattern (`parallel_pending` struct consumed between `open` and `start`) — specific to parallel index scan; would benefit from a scan-manager section.

These are left for future ingests to pick up.

## Branch-WIP companion pages

Three pages describing PR #7062 branch state were filed on 2026-04-29 ahead of merge to capture knowledge surfaced while debugging the silent-miss in `sql/_19_apricot/_03_index_skip_scan/cases/_03_iss_700000.sql` and the parallel-index-on-partitioned-tables crash (CBRD-26722):

- [[components/parallel-list-scan]] — `input_handler_list` + `slot_iterator_list` design: static slice partitioning, lazy CAS-claimed membuf, per-page tfile tracking, the **TWO** silent-skip sentinels in the slot iterator (`-2` overflow filter + `tuple_count == 0` race window).
- [[flows/parallel-list-scan-open]] — end-to-end open sequence and the `run_jobs()` join-barrier happens-before that closes the silent-skip race.
- [[components/parallel-index-scan]] — `input_handler_index` + `slot_iterator_index` design plus the four-invariant CBRD-26722 fix (struct-layout superset + paired `offsetof` asserts; `manager::close()` canonical destruction; XASL-stream-frozen-at-compile-time + worker BTID restore via `m_input_handler->get_indx_info()`; final-iteration type roll-back to `S_INDX_SCAN` and dump-path OR pattern). Sourced from [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables|CBRD-26722 knowledge dump]].

All three pages are tagged `status: branch-wip` and will be reconciled into the canonical `parallel-scan-input-handler-{list,index}` / `parallel-scan-slot-iterator-{list,index}` pages on merge (per the Reconciliation Plan above).

> [!update] Post-write commits to this PR page (as of branch HEAD `0f8a107bb`)
> The "Deep analysis — supplementary findings" section above contains two claims that have been **superseded** by branch commits landing after this PR page was written (2026-04-24):
>
> 1. **"Worker 0 always becomes the membuf-worker"** — superseded by commit `0f8a107bb`. Membuf is now claimed lazily via `m_membuf_claimed.compare_exchange_strong` in `initialize()`; the first **live** worker through wins, regardless of slice index. An idx-0 worker that fails before `loop()` no longer strands the membuf. See [[components/parallel-list-scan]] § "Lazy membuf claim — CAS, not idx-bound".
> 2. **`m_tl_is_membuf_worker = (idx == 0 && m_has_membuf)` binding** — same supersession; the source no longer has this expression.
>
> A stale comment at `px_scan_input_handler_list.cpp:134` (`/* Phase 1: worker 0 drains membuf pages */`) survives in the live source and is a cleanup candidate — flagged in [[components/parallel-list-scan]].
>
> The `qfile_collect_list_sector_info` reuse path (commit `e51920146`) and the per-page-tfile pattern (commit `ddf2e3374`) are both reflected in the new branch-WIP pages.

### Commits after `c28c5945a` (snapshot baseline of original review doc)

The external code-review doc was originally written at HEAD `c28c5945a` and re-anchored to `0f8a107bb` on 2026-04-29. The 22 intervening commits group as follows:

**Greptile-review fixes (P1 static-analysis findings):**
- `243eed446` build error + greptile P1
- `c2db4b854` greptile P1 second pass
- `399dba587` greptile review (general)

**Fallback hardening — single-thread regression on missing parallel flag:**
- `8799d7778` INDEX spec NO_PARALLEL → single-thread fallback
- `ecfc6a6fe` LIST spec NO_PARALLEL → single-thread fallback (mirrors INDEX)
- `195155bc5` defer `pllsid_parallel` writes until `manager::open()` succeeds (null-deref window)
- `a869c28ec` initialize `null_hfid` / `null_oid` fully on parallel-list entry

**Two-parameter split + cost-gate:**
- `8dc998587` client-side parallel index degree + count-only — splits `parallel_index_scan_page_threshold` (32) from server-side `parallel_scan_page_threshold` (2048); adds 137-line `qo_apply_parallel_index_scan_threshold` in `plan_generation.c`
- `fe84e61e3` gate parallel index scan on btree file pages
- `80471464c` revert: drop `estimated_leaf_pages` cost-gate plumbing

**BUILDVALUE_OPT:**
- `c047445c8` reuse first-operand fetch in DISTINCT loop
- `e66051b9e` merge `parallel_buildvalue_heap` → 13-aggregate whitelist landed

**Bug fixes:**
- `80056ecd9` parallel aptr trace deletion (memory leak)

**Performance — eliminate redundant page fixes:**
- `e51920146` reuse `qfile_collect_list_sector_info` + per-page tfile (PR #6981 pattern integration)
- `ddf2e3374` eliminate double-fix in list scan
- `1dd52e1b8` eliminate double-fix in index scan

**Concurrency hardening:**
- `0f8a107bb` **HEAD** — latch-couple `px_scan` index descent (root→leaf + leaf chain) + harden parallel promotion path; lazy-CAS membuf claim decoupled from idx 0

**Style / merges (no semantic change):**
- `3c04243bd`, `f497ae291` codestyle
- `4de6bf219`, `d84b6d0b8` no-op / shell test fixup
- `fbc05d96e`, `6b1147a88`, `bc2306a9a` merge develop / CUBRID:develop into branch

### Commits after `0f8a107bb` (CBRD-26722 parallel-index-on-partitioned-tables)

Four follow-on commits between the `0f8a107bb` review snapshot and the current branch HEAD `7fdb82099` ship the parallel-index-scan-on-partitioned-tables feature. Each commit fixes a different invariant:

- `67e0eb852` **C1** — `PARALLEL_INDEX_SCAN_ID` reshaped as a layout superset of `INDX_SCAN_ID` (mirrored offsets + appended pisid-only fields), pinned with paired `offsetof` `static_assert`s in `scan_manager.h:170-313`. Fixes the union-flip-corruption hazard during parallel-promote ↔ partition-reopen sequences.
- `1867903c0` **C2** — guard at `px_scan.cpp:1319` relaxed to allow parallel index scan on partitioned tables; the parent-class first-call path remains short-circuited via the `curent==NULL` branch.
- `9185c1aae` **C3** — worker `task::initialize` (`px_scan_task.cpp:130-145`) overrides the cloned `spec->indexptr->btid` with the manager's per-partition BTID via `m_input_handler->get_indx_info()`. The XASL stream is compile-time-frozen, so the live in-memory `qexec_init_next_partition` BTID update at `query_executor.c:9073` is invisible to `clone_xasl`'d workers; without C3 the parent class root `pgbuf_fix` returns NULL with no `er_set`, surfacing as the wrapped `ER_PT_EXECUTE(-495)` at `qexec_execute_mainblock:16581`.
- `7fdb82099` **C4 (HEAD)** — `query_dump.c:3093, 3553` accept either `S_PARALLEL_INDEX_SCAN || S_INDX_SCAN` (mirrors the HEAP-side pattern at `:3540`). The final-iteration parent-class re-open at `qexec_init_next_partition` rolls type back to `S_INDX_SCAN` while pisid `trace_storage` (preserved by C1's superset layout) still holds populated stats; dump branches need the OR to emit the `parallel workers: N` line.

These four commits are the structural distillation captured at [[components/parallel-index-scan]] (branch-WIP) and the source dump at [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables]]. They affect only the parallel-index path and the partitioned-table dispatch; they do not change the Reconciliation Plan shape (the planned `parallel-scan-input-handler-index` and `parallel-scan-slot-iterator-index` pages will absorb the C1/C3 invariants on merge; the existing scan-manager and query-dump plan items absorb the C4 dump-branch widening; C2 is a guard relax with no wiki-side surface beyond the existing checker tables).

The Reconciliation Plan above remains valid — none of these post-`c28c5945a` commits introduce surface that the plan misses (the two-parameter split, latch-couple, lazy CAS, double-fix removal, and the four-commit CBRD-26722 fix are all already captured in the plan's per-page deltas via the [[components/parallel-list-scan]] / [[components/parallel-index-scan]] branch-WIP pages and the existing system-parameter / cost-gate / scan-manager / query-dump plan items).

## Pre-merge integration analysis

> [!note] 2026-04-28 — `qfile_collect_list_sector_info` integration decision-prep
> External session note ingested as [[sources/2026-04-28-tfile-role-analysis|tfile role analysis]] explores whether `px_scan_input_handler_list` (PR #7062 branch) should adopt the `qfile_collect_list_sector_info` + `tfiles[]` parallel-array pattern that landed in the now-merged [[prs/PR-6981-parallel-hash-join-sector-split|PR #6981]] (baseline `0be6cdf6`).
>
> The analysis isolates `tfile`'s actual role on the read/free path (membuf lookup + free-time page-type discrimination only) and concludes that **disk-page-only** scans can technically operate without per-sector `tfile` tracking — but recommends adopting PR #6981's full pattern (Option B) for code-deduplication and uniform ownership, with a possible reduction to Option B' (no per-sector tfile, only per-sector `list_id`) if the overflow-chain traversal in `qfile_get_tuple` doesn't observably need the dependent's tfile context. No component pages have been edited as a result of this analysis (PR #7062 is still OPEN); baseline-truth claims it surfaced have been applied incidentally to [[components/query-manager]] and [[components/list-file]].

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
