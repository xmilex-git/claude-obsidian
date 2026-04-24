---
type: pr
pr_number: 6911
pr_url: "https://github.com/CUBRID/cubrid/pull/6911"
repo: "CUBRID/cubrid"
state: merged
author: "xmilex-git"
merged_at: "2026-03-27T07:46:31Z"
merge_commit: "45730b9006ac0c3c93a15ec46fdb3e8a58d239c6"
base_ref: "develop"
head_ref: "ftab"
base_sha: "630fdc4118cb9199c4f32d81aaba287cd5a59909"
head_sha: "e75014a6bec392f03b23198402cece2656d7da40"
jira: "CBRD-26615"
files_changed:
  - "cubrid/CMakeLists.txt"
  - "src/query/parallel/px_heap_scan/px_heap_scan.cpp"
  - "src/query/parallel/px_heap_scan/px_heap_scan.hpp"
  - "src/query/parallel/px_heap_scan/px_heap_scan_ftab_set.hpp"
  - "src/query/parallel/px_heap_scan/px_heap_scan_input_handler.hpp"
  - "src/query/parallel/px_heap_scan/px_heap_scan_input_handler_ftabs.cpp"
  - "src/query/parallel/px_heap_scan/px_heap_scan_input_handler_ftabs.hpp"
  - "src/query/parallel/px_heap_scan/px_heap_scan_input_handler_single_table.cpp"
  - "src/query/parallel/px_heap_scan/px_heap_scan_task.hpp"
  - "src/storage/file_manager.c"
  - "src/storage/file_manager.h"
related_components:
  - "[[components/parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-input-handler]]"
  - "[[components/parallel-heap-scan-task]]"
  - "[[components/file-manager]]"
  - "[[components/heap-file]]"
  - "[[components/page-buffer]]"
related_sources:
  - "[[sources/cubrid-src-query-parallel]]"
  - "[[sources/cubrid-src-storage]]"
triggered_baseline_bump: false
baseline_before: "175442fc858bd0075165729756745be6f8928036"
baseline_after: "175442fc858bd0075165729756745be6f8928036"
ingest_case: "b-already-absorbed"
tags:
  - pr
  - cubrid
  - parallel-query
  - heap-scan
  - storage
  - performance
created: 2026-04-24
updated: 2026-04-24
status: merged
---

# PR #6911 — Reduce I/O bottleneck when parallel heap scan (CBRD-26615)

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **Merged:** 2026-03-27 by `@xmilex-git` · **Merge commit:** `45730b9`
> **Base → Head:** `develop` (`630fdc41`) → `ftab` (`e75014a6`)
> **Stats:** 11 files changed, 27 commits, 52 reviews, 30 review threads, +515/-229 lines

> [!note] Ingest classification: case (b) — already absorbed
> The merge commit `45730b9` is **36 commits behind** the current wiki baseline `175442fc8`. The tree state this PR produces is already reflected in the existing [[components/parallel-heap-scan]] family of wiki pages. This page is **retroactive documentation only** — no page reconciliation was performed and the baseline was not bumped.

## Summary

Eliminates the global-mutex bottleneck in the parallel heap-scan page-distribution loop by replacing per-page `page_next` handoff with **upfront sector allocation**. At scan start the main thread reads the heap file's `PART_FTAB` + `FULL_FTAB` headers once, builds a collector of every data-bearing 64-page sector, splits that collector by worker count, and hands each worker an independent sector range. Workers then walk their own bitmap and call `pgbuf_fix` without cross-thread coordination — I/O issue points are no longer serialized on a single mutex, and scan time on large heaps drops accordingly.

## Motivation

Jira ticket: [CBRD-26615](http://jira.cubrid.org/browse/CBRD-26615).

The pre-PR design (`px_heap_scan_input_handler_single_table.cpp`) used a shared input handler guarded by a global mutex. Every worker called `page_next` to receive the next page number, then issued `pgbuf_fix`. The mutex protected the shared iterator, so workers that hit I/O wait serialized against each other — scaling beyond ~2 workers gave almost no speedup on cold caches. From the PR description: *"스레드가 페이지를 할당받고 I/O를 수행하는 과정에서 동기화 호출이 잦아지고, 사실상 I/O를 기다리는 시점이 직렬화되어 다중 스레드의 이점을 충분히 활용하지 못하는 병목이 존재했습니다."*

## Changes

### Structural

**New files:**
- `src/query/parallel/px_heap_scan/px_heap_scan_ftab_set.hpp` (+99) — FTab-set abstraction. Holds the pre-computed collection of data-bearing sectors and the split logic that apportions them across workers.
- `src/query/parallel/px_heap_scan/px_heap_scan_input_handler_ftabs.cpp` (+173) — new input handler implementing sector-based iteration. Contains `init_on_main` (header fetch + sector split) and `get_next_vpid_with_fix` (per-worker local bitmap walk, no mutex).

**Deleted files:**
- `src/query/parallel/px_heap_scan/px_heap_scan_input_handler_single_table.cpp` (−109) — the old global-mutex handler.
- `src/query/parallel/px_heap_scan/px_heap_scan_input_handler.hpp` (−64) — the abstract base that the single-table variant extended; replaced by the concrete `_ftabs` handler directly.

### Storage layer (file_manager)

- **`file_manager.h` (+32):** declares `FILE_FTAB_COLLECTOR`, `FILE_FULL_PAGE_BITMAP` macros, and the public `file_get_all_data_sectors` interface.
- **`file_manager.c` (+167/−27):** implements `file_get_all_data_sectors`, which traverses both `PART_FTAB` (partially-used sectors) and `FULL_FTAB` (fully-allocated sectors), collecting all sector entries that contain actual data into the `FILE_FTAB_COLLECTOR`. Called once per parallel heap scan at scan start.

### Parallel heap-scan layer (px_heap_scan)

- **`px_heap_scan_input_handler_ftabs.cpp` `init_on_main`:** the main thread invokes `file_get_all_data_sectors`, then splits the collector into worker-owned chunks according to parallelism degree.
- **`px_heap_scan_input_handler_ftabs.cpp` `get_next_vpid_with_fix`:** each worker walks its own sector bitmap locally and calls `pgbuf_fix` directly — no global lock, no cross-thread handoff per page.
- **Wiring:** `px_heap_scan.cpp`/`px_heap_scan.hpp` switched from instantiating the single-table handler to the ftabs-based one; `px_heap_scan_task.hpp` signature adjusted accordingly.

### Behavioral

- **Invalidated pages are tolerated.** A worker may fix pages whose records were deleted after the sector collection; the author confirmed in review that this is safe because (a) records on those pages are all in a deleted state and will be filtered out, and (b) `pgbuf_fix` succeeds as long as the file itself has not been dropped.
- **Lifecycle guarantee.** `initialize` and `finalize` are always paired by the `task execute()` frame, so the per-worker `FILE_FTAB_COLLECTOR` allocations are always freed even on error (confirmed in review — `pgbuf_tracker` would trip a debug assertion otherwise).
- **Fix-before-unfix ordering retained.** For ordered-pagetype safety (`PAGE_HEAP` / `PAGE_OVERFLOW`), the worker fixes the next page before unfixing the current one — deadlock-prevention discipline carried over from the old handler, not redundant despite sector-based iteration (per author clarification to `@youngjinj`).

## Review discussion highlights

Three engineers provided substantive feedback; all threads resolved before merge.

- **`@hornetmj` — argument type generality.** Suggested changing `file_get_all_data_sectors`'s first parameter from `HFID` to `VFID` for reusability across non-heap file types. Applied.
- **`@hornetmj` / `@shparkcubrid` — page_buffer API design.** The PR initially added an `allow_not_ordered_page` bool to `pgbuf_ordered_fix` in `src/storage/page_buffer.c`. Reviewers pushed back, suggesting the caller should assert `PGBUF_IS_ORDERED_PAGETYPE` instead of widening the pgbuf API. After iteration the author restricted the new code path to unconditional latches and ultimately **reverted all `page_buffer.c` changes from the merged PR** — final diff does not touch `page_buffer.c`. The sector-based handler achieves safety without broadening the pgbuf interface.
- **`@youngjinj` — unnecessary cast.** `(input_handler_ftabs *)` cast no longer needed post-refactor; removed.
- **`@youngjinj` — pgbuf_ordered_fix conditionality.** Proposed checking `PGBUF_IS_ORDERED_PAGETYPE` at the call site rather than extending `pgbuf_ordered_fix`. Author agreed; see previous point — resolved by design change (no pgbuf API change at all).
- **`@youngjinj` — sector traversal semantics.** Asked whether fix-new-before-unfix-old ordering was still necessary when scans are no longer `next`-chained. Author retained the pattern for deadlock prevention (confirmed required even in the sector-independent case).

The review history shows a **design convergence**: the initial approach (extend `pgbuf_ordered_fix` with a new bool parameter) was rejected in favor of a narrower change (caller-side type checks + sector-based independence), resulting in zero `page_buffer.c` churn in the final merge.

## Wiki impact

### Reconciliation

**None performed** (case b — merge commit already absorbed by baseline `175442fc8`). The existing wiki pages below already describe the post-PR state:

- [[components/parallel-heap-scan]] — parent component
- [[components/parallel-heap-scan-input-handler]] — this is the `_ftabs` handler introduced by this PR (the page exists post-merge, reflecting the final state)
- [[components/parallel-heap-scan-task]]
- [[components/file-manager]] — contains `file_get_all_data_sectors`
- [[components/heap-file]]
- [[sources/cubrid-src-query-parallel]] — source-page summary; mentions the `_ftabs` variant
- [[sources/cubrid-src-storage]]

### Baseline bump

**None** (case b).
- Before: `175442fc858bd0075165729756745be6f8928036`
- After:  `175442fc858bd0075165729756745be6f8928036` (unchanged)

### Decision-record candidate

The review-driven reversal of the `pgbuf_ordered_fix(..., allow_not_ordered_page)` widening is a small but clean ADR candidate — a case of "interface stayed narrow because caller-side assertions were sufficient." Not filing it automatically, flagging for the user's judgment. Anchor: `decisions/NNNN-pgbuf-narrow-over-allow-not-ordered.md` if pursued.

## Related

- [[prs/_index|PRs]]
- CUBRID upstream: https://github.com/CUBRID/cubrid/pull/6911
- Jira: CBRD-26615
- Components: [[components/parallel-heap-scan]], [[components/file-manager]]
- Source summaries: [[sources/cubrid-src-query-parallel]], [[sources/cubrid-src-storage]]
