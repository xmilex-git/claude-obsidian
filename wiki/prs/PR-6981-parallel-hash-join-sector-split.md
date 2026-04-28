---
type: pr
pr_number: 6981
pr_url: "https://github.com/CUBRID/cubrid/pull/6981"
repo: "CUBRID/cubrid"
state: MERGED
is_draft: false
author: "youngjinj"
created_at:
merged_at: "2026-04-27T14:29:23Z"
closed_at: "2026-04-27T14:29:23Z"
merge_commit: "0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0"
base_ref: "develop"
head_ref: "CBRD-26666"
base_sha: "cc563c7fd90521393781d8440bf5144d2566ff71"
head_sha: "2277b3d882f3565141a1d4786cae70f45ef0a325"
jira: "CBRD-26666"
files_changed:
  - "src/query/list_file.c (+127/-0)"
  - "src/query/list_file.h (+4/-0)"
  - "src/query/parallel/px_hash_join/px_hash_join.cpp (+21/-5)"
  - "src/query/parallel/px_hash_join/px_hash_join_task_manager.cpp (+155/-89)"
  - "src/query/parallel/px_hash_join/px_hash_join_task_manager.hpp (+21/-0)"
  - "src/query/query_hash_join.c (+27/-6)"
  - "src/query/query_hash_join.h (+6/-6)"
  - "src/query/query_list.h (+23/-0)"
related_components:
  - "[[components/parallel-hash-join]]"
  - "[[components/parallel-hash-join-task-manager]]"
  - "[[components/list-file]]"
  - "[[components/file-manager]]"
related_sources:
  - "[[sources/cubrid-src-query]]"
  - "[[sources/cubrid-src-query-parallel]]"
ingest_case: c
triggered_baseline_bump: true
baseline_before: "cc563c7fd90521393781d8440bf5144d2566ff71"
baseline_after: "0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0"
reconciliation_applied: true
reconciliation_applied_at: 2026-04-28
incidental_enhancements_count: 1
tags:
  - pr
  - cubrid
  - parallel-query
  - hash-join
  - lock-free
  - sector
  - merged
created: 2026-04-28
updated: 2026-04-28
status: merged
---

# PR #6981 — Improve parallel hash join split phase with sector-based page distribution

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `MERGED` · **Author:** `@youngjinj` · **Merge commit:** `0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0`
> **Base → Head:** `develop` (`cc563c7f`) → `CBRD-26666` (`2277b3d8`)

> [!note] Ingest classification: case (c) — newer than baseline
> Merge commit `0be6cdf6` is a direct child of the prior baseline `cc563c7f` on `develop` (single squash-merge). PR-reconciliation applied immediately + baseline bumped.

## Summary

Replaces the parallel hash-join *split* phase's `scan_mutex`-serialised page handoff with a lock-free sector-bitmap distribution. Workers now claim disk sectors via `std::atomic<int> next_sector_index` and walk each 64-page bitmap with `__builtin_ctzll` to compute VPIDs directly, eliminating per-page mutex contention. Membuf pages are claimed by exactly one worker via a CAS on `std::atomic<bool> membuf_claimed`. A new generic helper `qfile_collect_list_sector_info` (in `list_file.c`) harvests sectors from a `QFILE_LIST_ID` and every page in its `dependent_list_id` chain into a single flat `QFILE_LIST_SECTOR_INFO` (sectors + parallel `tfiles[]` array) that the workers share.

The PR also fixes two latent correctness issues uncovered while restructuring: overflow continuation pages now follow the `m_current_tfile` recorded by the page owner (previously `list_id->tfile_vfid` was used unconditionally, which is wrong for pages from dependent list_ids); and the merge-phase partition flush now uses `qfile_truncate_list` + retained `LIST_ID` instead of `qfile_destroy_list` + free, so a mid-merge `qfile_append_list` failure can't leave a half-freed `LIST_ID` for the cleanup path to double-free.

## Motivation

The previous split-phase design (introduced when parallel hash join landed) had every worker contend on a single `HASHJOIN_SHARED_SPLIT_INFO::scan_mutex` to advance a shared `(scan_position, next_vpid)` cursor one page at a time:

```cpp
std::lock_guard<std::mutex> lock (m_shared_info->scan_mutex);
switch (m_shared_info->scan_position) {
  case S_BEFORE:  /* fetch first_vpid */
  case S_ON:      /* QFILE_GET_NEXT_VPID(page) → next_vpid */
  case S_AFTER:   /* return nullptr */
}
```

This serialises *every* `get_next_page` call across all workers, so wall-clock split throughput is bounded by mutex hold time rather than scaling with `num_parallel_threads`. The same pattern was already replaced in the parallel heap-scan path (PR #6911 — sector pre-split via `file_get_all_data_sectors`); this PR brings the same primitive to parallel hash join's input list scan.

The author's secondary goal is reusing the existing `FILE_PARTIAL_SECTOR` / `FILE_FTAB_COLLECTOR` infrastructure rather than introducing a new distribution scheme. Membuf pages cannot be sector-distributed (they live in memory, with `volid = NULL_VOLID`), so a CAS-claim path keeps a single owner walking them sequentially.

## Changes

### Structural

**New types (in `query_list.h`):**
- `struct qfile_list_sector_info` / `QFILE_LIST_SECTOR_INFO` — the cross-list sector flat-array.
  - `struct qmgr_temp_file *membuf_tfile` — tfile owning membuf pages (NULL when no membuf in the chain). Membuf only exists in the *first* `QFILE_LIST_ID`, never in dependents.
  - `struct file_partial_sector *sectors` — concatenated sector array across base + dependent list ids. Heap-allocated via `db_private_alloc/realloc`.
  - `void **tfiles` — parallel array, one entry per sector, holding the `QMGR_TEMP_FILE *` that owns that sector. Required because dependent-list pages must be released against their own tfile, not the base list's tfile.
  - `int sector_cnt`.
- `QFILE_LIST_SECTOR_INFO_INITIALIZER` and `QFILE_INIT_LIST_SECTOR_INFO(info)` macros.

**New API (in `list_file.c` / `list_file.h`):**
- `int qfile_collect_list_sector_info (THREAD_ENTRY *, QFILE_LIST_ID *list_id, QFILE_LIST_SECTOR_INFO *out)` — sets `membuf_tfile` from the *first* list_id (only when `membuf_last >= 0` AND `membuf != NULL`), then walks the `dependent_list_id` chain and calls `file_get_all_data_sectors` on each `temp_vfid` (skipping `VFID_ISNULL` entries — purely-membuf lists). Sectors and the parallel `tfiles[]` array grow via paired `db_private_realloc`. The function calls itself's free at entry, so it is safe to reuse a `QFILE_LIST_SECTOR_INFO` across outer/inner phases without explicit reset (the `px_hash_join.cpp` outer→inner sequence relies on this).
- `void qfile_free_list_sector_info (THREAD_ENTRY *, QFILE_LIST_SECTOR_INFO *)` — frees both `sectors` and `tfiles`, NULLs `membuf_tfile`, zeros `sector_cnt`.

**Revised `HASHJOIN_SHARED_SPLIT_INFO` (in `query_hash_join.h`):**
- Removed: `std::mutex scan_mutex`, `SCAN_POSITION scan_position`, `VPID next_vpid`.
- Added: `QFILE_LIST_SECTOR_INFO sector_info`, `std::atomic<bool> membuf_claimed` (default `false`), `std::atomic<int> next_sector_index` (default `0`).
- `part_mutexes` retained — per-partition list-file lock is unaffected by this PR.

**Revised `split_task` per-thread state (in `px_hash_join_task_manager.hpp`):**
- New header forward-declaration `struct qmgr_temp_file` + `typedef … QMGR_TEMP_FILE` so the private member can be a `QMGR_TEMP_FILE*` without including `query_manager.h`.
- New private members: `int m_membuf_index` (`-1` = not the membuf owner), `int m_sector_index` (`-1` = need next sector), `UINT64 m_current_bitmap`, `VSID m_current_vsid`, `QMGR_TEMP_FILE *m_current_tfile`.

### Per-file notes

- `src/query/query_list.h` — new struct + initializer + init-macro ([[components/list-file]]).
- `src/query/list_file.c`, `list_file.h` — `qfile_collect_list_sector_info` + `qfile_free_list_sector_info` ([[components/list-file]]).
- `src/query/query_hash_join.h` — `HASHJOIN_SHARED_SPLIT_INFO` field swap (mutex/cursor → sector_info/atomics) ([[components/parallel-hash-join]]).
- `src/query/query_hash_join.c` — `hjoin_init_shared_split_info` adds two atomic-default asserts; `hjoin_clear_shared_split_info` calls `qfile_free_list_sector_info` *before* the `part_cnt <= 1` early-return (commented in code: "must be a separate call, not in the (part_cnt > 1) branch"). The serial fallback `hjoin_split_qlist` also gets the `qfile_destroy_list+QFILE_FREE_AND_INIT_LIST_ID → qfile_truncate_list` correctness fix (mirrored to the parallel path).
- `src/query/parallel/px_hash_join/px_hash_join.cpp` — `build_partitions` calls `qfile_collect_list_sector_info` before each split phase (outer, then inner) and resets the two atomics; cleanup goes through `hjoin_clear_shared_split_info` which now frees the sector_info ([[components/parallel-hash-join]]).
- `src/query/parallel/px_hash_join/px_hash_join_task_manager.{hpp,cpp}` — `split_task` ctor inits the 5 new members; `get_next_page` rewritten end-to-end (see below); `execute()` switches all `qmgr_get_old_page` / `qmgr_free_old_page_and_init` calls for the data + overflow chain from `list_id->tfile_vfid` to `m_current_tfile`; partition-flush merge restructured to use `qfile_truncate_list`-on-success and a separate-`if` (not `else`) cleanup branch ([[components/parallel-hash-join-task-manager]]).

### Behavioral

- **Lock-free split-phase distribution.** `split_task::get_next_page` no longer takes any mutex. Phase 1 (membuf): exactly one worker wins `membuf_claimed.compare_exchange_strong (false → true, acq_rel)` and walks `m_membuf_index` from `0` to `membuf_tfile->membuf_last` inclusive, with `vpid = {NULL_VOLID, m_membuf_index}` for each fetch. Non-winners skip Phase 1 entirely. Phase 2 (sector): each worker pulls a fresh sector via `next_sector_index.fetch_add(1, relaxed)`, copies `sectors[sector_index].vsid` and `.page_bitmap` into per-thread `m_current_vsid` / `m_current_bitmap`, then iterates set bits with `__builtin_ctzll` (or `bit64_count_trailing_zeros` on non-GCC/Clang), computing `vpid.pageid = SECTOR_FIRST_PAGEID(vsid.sectid) + bit_pos`.
- **Per-worker tfile tracking.** `m_current_tfile` is recorded by `get_next_page` on every successful return (membuf path → `sector_info->membuf_tfile`; sector path → `sector_info->tfiles[m_sector_index]`). All subsequent `qmgr_free_old_page_and_init` calls and the overflow-chain `qmgr_get_old_page`/`qmgr_free_old_page_and_init` calls in `execute()` use `m_current_tfile`, not `list_id->tfile_vfid`. This is required for correctness whenever the input list has dependents — the prior code would have passed the *base-list* tfile when freeing a *dependent-list* page, breaking the temp-file allocator's bookkeeping.
- **Overflow continuation pages skipped on bitmap walk.** A page with `QFILE_GET_TUPLE_COUNT (page) == QFILE_OVERFLOW_TUPLE_COUNT_FLAG` (`-2`) is now released and the loop continues. These continuation pages share the start page's sector bitmap (they are allocated normally, not in a separate file), so a naive bitmap walker would otherwise consume them twice — once via the bitmap and once via the start-page owner's `QFILE_GET_OVERFLOW_VPID` chain.
- **Membuf NULL guard.** `qfile_collect_list_sector_info` requires `membuf != NULL` *and* `membuf_last >= 0` before setting `membuf_tfile`. Without the NULL guard a `FILE_QUERY_AREA` result file (which can have `membuf_last >= 0` but `membuf == NULL`) would SEGFAULT in Phase 1 — see Remarks in PR body.
- **Outer→inner reuse.** `qfile_collect_list_sector_info` calls its own free at entry, so `build_partitions` does not need an explicit free between the outer-split and inner-split rounds. Cleanup at the end of `build_partitions` happens through `hjoin_clear_shared_split_info`, which itself calls `qfile_free_list_sector_info` *before* the `part_cnt <= 1` early-return (the inline code comment specifically warns against moving the free into the `(part_cnt > 1)` branch — that would leak when `part_cnt <= 1`).
- **Partition merge correctness fix.** Both `hjoin_split_qlist` (serial fallback) and `split_task::execute` (parallel) used to call `qfile_destroy_list (temp_part_list_id[part_id])` + `QFILE_FREE_AND_INIT_LIST_ID (temp_part_list_id[part_id])` in the success path *unconditionally*, including before the `qfile_append_list` return value was checked. After the PR, `qfile_append_list`'s return is checked first; on success the pages are merged into the shared partition and the temp list_id is *truncated* (pages reclaimed, descriptor preserved) rather than destroyed. The structural rewrite splits the post-loop cleanup into two separate `if` blocks (one for `!has_error`, one for `has_error`) instead of an `if/else`, with a comment in the parallel path: *"must be a separate `if`, not an `else` of the block above: the merge loop above may set has_error = true via break, and that case still needs this cleanup to run."*

### New surface (no existing wiki reference)

- `qfile_collect_list_sector_info` / `qfile_free_list_sector_info` / `QFILE_LIST_SECTOR_INFO` — captured on [[components/list-file]] under "Sector-based page distribution".
- The `m_current_tfile` worker-state pattern — captured on [[components/parallel-hash-join-task-manager]] under "Page Cursor — Lock-free sector bitmap walk".

## Review discussion highlights

29 reviews, 20 issue-comments, ~10 inline review comments. Substantive design rationale (filtering nits and bot output):

- **`qfile_truncate_list` was author-introduced** as part of the PR rather than reused — the prior `qfile_destroy_list` path had no failure-aware variant. Multiple reviewers flagged the `qfile_destroy_list+free → truncate+keep` change as a separable correctness fix and asked whether it should also be applied to the serial path; the author applied it to `hjoin_split_qlist` (`query_hash_join.c`) in the same PR.
- **Why a single CAS instead of a counter for membuf.** Reviewer asked whether membuf could be split across workers like sectors. Author: membuf is a contiguous in-memory page array with no bitmap or index abstraction — a single owner walking it linearly is simpler and avoids per-page atomic overhead, and the membuf budget is small relative to disk-spilled portions where the parallelism payoff is.
- **`QFILE_LIST_SECTOR_INFO::tfiles` typed `void **` rather than `QMGR_TEMP_FILE **`.** The struct lives in `query_list.h` (shared with C); `QMGR_TEMP_FILE` is forward-declared via `struct qmgr_temp_file` and the typedef sits in `query_manager.h`. Keeping the field `void **` avoids pulling `query_manager.h` into the public header. Workers cast at use site (`(QMGR_TEMP_FILE *) tfiles[m_sector_index]`).
- **Memory order choices.** `membuf_claimed` uses `acq_rel` for the CAS (the winner publishes ownership before walking; losers acquire the established state). `next_sector_index` uses `relaxed` for both the initial `store(0)` and per-claim `fetch_add(1)` — sectors are independent, no payload synchronisation crosses the counter.
- **No backwards-compat shim for `S_BEFORE`/`S_ON`/`S_AFTER`.** The `SCAN_POSITION` enum is defined elsewhere (`storage_common.h`) and used widely — `query_hash_join.h` simply stops referencing it. `px_hash_join.cpp` still includes `storage_common.h` for unrelated VPID macros.

## Reconciliation Plan

Applied during this ingest — see Pages Reconciled below.

## Pages Reconciled

All four pages received `[!update]` callouts citing **PR #6981 + merge sha `0be6cdf6`**:

- [[components/parallel-hash-join-task-manager]] — major rewrite: `split_task` row in method table updated (`get_next_page` description + new per-thread fields); "Execution Path — `split_task::execute`" diagram updated (no `scan_mutex` lock per page; `m_current_tfile` recorded); "Page Cursor State Machine" section retitled "Page Cursor — Lock-free sector bitmap walk" with two-phase membuf/sector pseudocode and `m_current_tfile` recording; new key-insight "Lock-free split distribution"; "Constraints — Threading" updated to remove `scan_mutex` from the split path.
- [[components/parallel-hash-join]] — `key_files` line for `px_hash_join_task_manager` mentions sector distribution; "Constraints — Threading" updated (no `scan_mutex` in split phase; `sector_info` + atomics distribute pages); `build_partitions` flow updated to mention `qfile_collect_list_sector_info` calls.
- [[components/list-file]] — new section "Sector-based page distribution (`qfile_collect_list_sector_info`)" documenting the helper, the `QFILE_LIST_SECTOR_INFO` struct, the dependent-list-chain merge, the membuf NULL guard, and the parallel `tfiles[]` array; cross-link from "Dependent-list chain" subsection.
- [[components/file-manager]] — `[!update]` under "Data-sector harvesting" adding *parallel hash-join split phase* to the consumer list (alongside parallel heap scan, parallel list scan, `SORT_INDEX_LEAF`).

## Incidental wiki enhancements

- [[components/list-file]] — `[!gap]` filled: prior page mentioned `QFILE_OVERFLOW_TUPLE_COUNT_FLAG` skipping but only in the dataflow context; the PR analysis surfaced the **exact bitmap-double-consumption** mechanism (continuation pages share the start page's sector bitmap because `qfile_allocate_new_ovf_page` allocates from the same temp file). Added a sentence under "QFILE_OVERFLOW_TUPLE_COUNT_FLAG" tying the marker to sector-bitmap walkers explicitly. Counted as 1 incidental enhancement.

## Baseline impact

- Before: `cc563c7fd90521393781d8440bf5144d2566ff71`
- After:  `0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0`
- Bump triggered: `true`
- Logged: [[log]] under `[2026-04-28] baseline-bump | cc563c7f → 0be6cdf6 (PR #6981)`

## Related

- [[prs/_index|PRs]]
- [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911]] — the sibling design that introduced sector pre-split for parallel heap scan; this PR ports the same primitive to parallel hash join's input list scan.
- [[prs/PR-7011-parallel-index-build|PR #7011]] — also reuses `file_get_all_data_sectors` (for `SORT_INDEX_LEAF`).
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/6981
- Jira: http://jira.cubrid.org/browse/CBRD-26666
