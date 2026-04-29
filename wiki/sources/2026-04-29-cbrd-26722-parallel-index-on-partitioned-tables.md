---
address: c-000007
type: source
source_path: "/home/cubrid/dev/cubrid/.claude/wiki-updates/CBRD-26722-parallel-index-on-partitioned-tables.md"
source_branch: parallel_scan_all
source_head: "7fdb82099ce3e30b47deb9e5036e02fabddc351c"
source_hash: "4ba3c2bee6350fdcd767657d0b7414a1"
source_kind: knowledge-dump
ingested: 2026-04-29
related_pr: 7062
jira: CBRD-26722
tags:
  - source
  - cubrid
  - parallel-query
  - parallel-index-scan
  - partition
  - btree
  - knowledge-dump
  - branch-wip
related:
  - "[[components/parallel-index-scan]]"
  - "[[components/parallel-list-scan]]"
  - "[[components/parallel-heap-scan]]"
  - "[[components/scan-manager]]"
  - "[[components/btree]]"
  - "[[components/xasl]]"
  - "[[components/partition-pruning]]"
  - "[[components/query-executor]]"
  - "[[prs/PR-7062-parallel-scan-all-types|PR #7062]]"
created: 2026-04-29
updated: 2026-04-29
---

# Source: CBRD-26722 — Parallel Index Scan on Partitioned Tables (knowledge dump)

> [!info] Source provenance
> External knowledge-dump file authored on the CUBRID workspace branch `parallel_scan_all` at HEAD `7fdb82099` (4 commits ahead of the PR #7062 review snapshot `0f8a107bb`). Captures the analysis-loop history of the parallel-index-on-partitioned-table fix immediately after it shipped on the branch.
>
> Path: `/home/cubrid/dev/cubrid/.claude/wiki-updates/CBRD-26722-parallel-index-on-partitioned-tables.md`
> md5: `4ba3c2bee6350fdcd767657d0b7414a1`

## Why this source matters

Parallel index scan was unconditionally disabled on partitioned tables. Naive guard relaxation crashed with `ER_PT_EXECUTE(-495)` and an empty underlying error. The shipped fix is **four interlocking commits** because the bug crossed four different invariants, each unrelated to the others. Documenting them together is what kept future readers from re-deriving the four-way convergence from cold.

The structural distillation lives at [[components/parallel-index-scan]] (branch-WIP). This page is the source-trail.

## Four-commit fingerprint

```
67e0eb852  C1  PARALLEL_INDEX_SCAN_ID superset of INDX_SCAN_ID (struct layout)
1867903c0  C2  Relax guard at px_scan.cpp:1319 (parent-class only) (type-machine state)
9185c1aae  C3  Restore partition BTID in worker initialize (XASL clone boundary)
7fdb82099  C4  Dump parallel-index trace_storage on S_INDX_SCAN type (dump-path discrimination)
```

## Five facts captured

1. **`SCAN_ID::s` union flips and the layout-mismatch failure mode.** The 3-field `parallel_index_scan_id` shared storage with the ~50-field `indx_scan_id`. Each parallel-on-partition flip corrupted the first 24 bytes of isid. `PARALLEL_HEAP_SCAN_ID` doesn't suffer this because it's a layout superset of `HEAP_SCAN_ID`. C1 reshapes pisid the same way and pins the invariant with paired `offsetof` asserts.

2. **`manager::close()` is the canonical destruction site.** Asymmetric vs. RAII C++ expectations: `close()` does destructor + free. Calling `~manager()` after `close()` is a double-free; the dispatch chain at `qexec_init_next_partition:8795` already handles partition-transition destruction.

3. **XASL stream carries compile-time BTID; per-partition runtime updates don't propagate to clones.** Main-thread `qexec_init_next_partition` (`query_executor.c:9073`) updates `spec->indexptr->btid` per partition before opening the scan — works for the main thread but parallel workers do `clone_xasl(thread_ref)` which re-deserializes from the stream and gets the parent class's BTID. `pgbuf_fix(Root_vpid)` returns NULL on the parent class root with no `er_set` → silent failure → `ER_PT_EXECUTE(-495)` wrapper at `qexec_execute_mainblock:16581`. C3 overrides the worker's cloned BTID via `m_input_handler->get_indx_info()->btid`.

4. **Latch-couple commit deferred root-descent: `m_btid_int.sys_btid` is NULL until first worker descends.** PR #7062 commit `0f8a107bb` made `init_on_main` minimal — only the BTID copy + indx_info store. The actual root descent moved to the lazy `descend_to_first_leaf` called by the first live worker. Consequence for tooling: `m_btid` and `get_indx_info()` are safe post-promote; `m_btid_int.sys_btid` is NOT.

5. **Parent-class re-open rolls type back to `S_INDX_SCAN`; dump path must accept both types.** Final-iteration semantics of `qexec_init_next_partition` (after last partition, `spec->curent == NULL`, parent BTID re-set, `scan_open_index_scan:3095` resets `scan_id->type = S_INDX_SCAN`). By the time `query_dump` runs, type is `S_INDX_SCAN`, not `S_PARALLEL_INDEX_SCAN`. The pisid superset (C1) keeps `pisid.trace_storage` valid through the flip; the dump branches now accept either type — C4 mirrors the existing HEAP-side OR pattern at `query_dump.c:3540`.

## Cross-cutting: trace_storage life-cycle leak

Per-partition orphan at `scan_try_promote` line 1560 (`scan_id->s.pisid.trace_storage = nullptr`). Documented as a known cosmetic limitation — aggregate SCAN-level counts and PARTITION lines are correct; the `parallel workers: N` line shows MIN..MAX across the LAST partition's workers only.

## Diagnosis playbook captured

The dump records the multi-round debug pattern:

- `ER_PT_EXECUTE(-495)` wrapper at `qexec_execute_mainblock:16581` fires when `stat != NO_ERROR && er_errid() == NO_ERROR`.
- The actual error was `er_clear()`'d upstream. Common sites: `scan_try_promote_parallel_index_scan` promote-fail (`px_scan.cpp:1547`) → fall back to `S_INDX_SCAN`; many in `query_executor.c`.
- File-based `fprintf` logging is the diagnostic tool of choice — server stderr is captured but not always visible; `er_set` doesn't fire so the server log is empty.
- For silent NULL returns from `pgbuf_fix`, check page-buffer mode mismatch or invalid page/file VPID.

## Acceptance criteria recorded (`7fdb82099`)

```
P region parallel workers: 5    (G1: ≥5)
P region PARTITION lines:  30   (G2: per-surviving-partition)
P/P-cmp result match:      P1✓ P2✓ P3✓ P4✓ P5✓
H region parallel workers: 2    (regression: HEAP partitioned still works)
L region parallel workers: 4    (regression: TEMP)
I region parallel workers: 8    (regression: INDEX non-partition)
F region parallel workers: 0    (correctly serial fallback)
ER_PT_EXECUTE(-495):       0
```

## File map cited (branch HEAD `7fdb82099`)

- `src/query/scan_manager.h:170-313` — `parallel_index_scan_id` superset + paired `offsetof` asserts (C1)
- `src/query/parallel/px_scan/px_scan.cpp:1319-1322` — guard relax (C2)
- `src/query/parallel/px_scan/px_scan.cpp:2169-2187` — `manager::close()`
- `src/query/parallel/px_scan/px_scan_input_handler_index.cpp:42-55` — minimal `init_on_main`
- `src/query/parallel/px_scan/px_scan_input_handler_index.cpp:56-90` — lazy `descend_to_first_leaf`
- `src/query/parallel/px_scan/px_scan_task.cpp:130-145` — worker BTID restore (C3)
- `src/query/query_executor.c:9073` — per-partition BTID write
- `src/query/query_executor.c:8794-8847` — partition transition + `set_last_partition_stats`
- `src/query/query_dump.c:3093, 3553` — dump branches accept `S_PARALLEL_INDEX_SCAN || S_INDX_SCAN` (C4)

## Disposition

- All five facts have been distilled into [[components/parallel-index-scan]] (`status: branch-wip`).
- [[prs/PR-7062-parallel-scan-all-types]] re-anchored to HEAD `7fdb82099` (was `0f8a107bb`); the four follow-on commits added to the "Commits after `c28c5945a`" log.
- No baseline bump (PR #7062 is OPEN).
- No incidental enhancements applied to baseline component pages this round — every fact in the dump is intertwined with the branch-only `px_scan/` directory or the latch-couple deferred descent (also branch-only). Baseline-truth surface remains drained from the prior 2026-04-29 round.

## Related

- Distilled: [[components/parallel-index-scan]]
- Tracking PR: [[prs/PR-7062-parallel-scan-all-types|PR #7062]]
- Sibling source: [[sources/2026-04-28-tfile-role-analysis]] (PR #7062 list-scan integration analysis)
- Sibling branch-WIP: [[components/parallel-list-scan]], [[flows/parallel-list-scan-open]]
- Underlying machinery: [[components/scan-manager]], [[components/xasl]], [[components/btree]], [[components/partition-pruning]], [[components/query-executor]]
- Jira: CBRD-26722
