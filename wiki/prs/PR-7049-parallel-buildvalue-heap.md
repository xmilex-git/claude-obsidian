---
type: pr
pr_number: 7049
pr_url: "https://github.com/CUBRID/cubrid/pull/7049"
repo: "CUBRID/cubrid"
state: MERGED
is_draft: false
author: "xmilex-git"
created_at: "2026-04-22"
merged_at: "2026-04-27T04:52:40Z"
closed_at: "2026-04-27T04:52:40Z"
merge_commit: "65d6915437eb6217ab0050939c6ad63f0d509735"
base_ref: "develop"
head_ref: "parallel_buildvalue_heap"
base_sha: "2be90e6ddfa9893cf1bc6952d8cb8f0f84684fc4"
head_sha: "c047445c888e6e8d4f414269520215fa634c394c"
jira: "CBRD-26711"
files_changed:
  - "src/query/parallel/px_heap_scan/px_heap_scan.cpp (rename only, 38 LOC)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_checker.cpp (+helper, 65 LOC)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_result_handler.cpp (+295/-148, the substantive change)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_result_handler.hpp (rename only)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_result_type.hpp (rename only)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_task.cpp (renames + 4-line err prop, 22 LOC)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_trace_handler.cpp (rename + label, 4 LOC)"
  - "src/query/query_executor.c (rename only, 4 LOC)"
  - "src/query/xasl.h (ACCESS_SPEC_FLAG rename, 2 LOC)"
related_components:
  - "[[components/parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-result-handler]]"
  - "[[components/parallel-heap-scan-task]]"
  - "[[components/parallel-heap-scan-support]]"
  - "[[components/aggregate-analytic]]"
  - "[[components/xasl-aggregate]]"
  - "[[components/query-executor]]"
related_sources:
  - "[[sources/cubrid-src-query-parallel]]"
  - "[[sources/cubrid-src-query]]"
  - "[[sources/cubrid-src-xasl]]"
ingest_case: c
triggered_baseline_bump: true
baseline_before: "175442fc858bd0075165729756745be6f8928036"
baseline_after: "65d6915437eb6217ab0050939c6ad63f0d509735"
reconciliation_applied: true
reconciliation_applied_at: "2026-04-27"
incidental_enhancements_count: 2
tags:
  - pr
  - cubrid
  - parallel-query
  - parallel-heap-scan
  - aggregation
  - buildvalue
  - merged
created: 2026-04-27
updated: 2026-04-27
status: merged
---

# PR #7049 — Support avg, sum function on parallel heap scan

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `MERGED` · **Author:** `@xmilex-git` · **Jira:** [CBRD-26711](https://jira.cubrid.org/browse/CBRD-26711)
> **Base → Head:** `develop` (`2be90e6dd`) → `parallel_buildvalue_heap` (`c047445c8`) · **Merge commit:** `65d691543` (2026-04-27 04:52 UTC)
> **Scale:** 9 files, +424 / −158 (582 LOC). Largest: `px_heap_scan_result_handler.cpp` at 443 changed LOC. 13 commits (squash-merged).
> **Reviews:** 17 inline comments (mostly greptile-apps bot P1/P2 — addressed via 6 "greptile review apply" commits + a final "Apply review feedback: reuse first-operand fetch in DISTINCT loop").

> [!note] Ingest classification: case (c) — newer than baseline
> Merge commit `65d691543` is a direct child of baseline `175442fc8` on `develop`. Full PR-reconciliation applied during this ingest; baseline bumped from `175442fc` → `65d69154`.

## Summary

Extends parallel heap scan's "fast path" for single-tuple aggregation (BUILDVALUE_PROC, i.e. aggregate without GROUP BY) from **just `COUNT(*)` and `COUNT(col)`** to the full set of order-independent aggregates: **`MIN`, `MAX`, `SUM`, `AVG`, `STDDEV*`, `VARIANCE*` plus their `DISTINCT` variants**. Each worker thread independently computes a partial aggregate in heap 0 (process-wide); the main thread merges all partials via `qdata_aggregate_accumulator_to_accumulator`, then re-clones the merged values into its own private heap so downstream `pr_clear_value` cleanup works correctly.

Cosmetically, the entire `RESULT_TYPE::COUNT_DISTINCT` enum value (and its sibling `ACCESS_SPEC_FLAG_COUNT_DISTINCT`, the `count_opt` local, the trace label `"count"`) is renamed to `BUILDVALUE_OPT` (`ACCESS_SPEC_FLAG_BUILDVALUE_OPT`, `buildvalue_opt`, `"buildvalue"`) to reflect the broadened scope — the COUNT-only naming was misleading after this PR. Eight of the nine touched files are pure rename + minor wiring; the engineering content lives in `px_heap_scan_result_handler.cpp`.

## Motivation

Pre-PR behavior: a query like `SELECT SUM(amount) FROM big_table` over a parallelizable heap is forced into either `MERGEABLE_LIST` mode (per-thread list of all qualifying rows, then aggregate sequentially in main) or `XASL_SNAPSHOT` mode (row-by-row handoff to main thread for aggregation), both incurring per-tuple cross-thread coordination. Only `COUNT(*)` and `COUNT(col)` benefited from the fast `COUNT_DISTINCT` mode where each worker independently maintained a partial counter and the main thread merely summed N integers at the end.

The whole class of order-independent associative aggregates (MIN, MAX, SUM, AVG, STDDEV*, VARIANCE*) shares the same parallelizability property — there's no fundamental reason workers couldn't accumulate partial values and the main thread combine them. CBRD-26711 closes that gap. Practical effect: `SELECT AVG(...) FROM <large_table>` queries that previously fell back to `MERGEABLE_LIST` (with full row materialization to per-thread lists) now use the fast partial-aggregate path with O(parallelism) merge cost instead of O(rows).

## Changes

### Structural

- **Enum rename** in `px_heap_scan_result_type.hpp`: `RESULT_TYPE::COUNT_DISTINCT` (0x3) → `RESULT_TYPE::BUILDVALUE_OPT` (0x3, value preserved). Comment changed from `/* (fast) for update statistics */` to `/* (fast) buildvalue proc aggregate optimization */`.
- **ACCESS_SPEC_FLAG rename** in `xasl.h`: `ACCESS_SPEC_FLAG_COUNT_DISTINCT` (`0x1 << 4`) → `ACCESS_SPEC_FLAG_BUILDVALUE_OPT`. Bit position preserved — wire-compatible if the flag isn't marshalled across versions (it isn't; client-side only).
- **New helper** in `px_heap_scan_checker.cpp` (file-static): `is_buildvalue_opt_supported_function(FUNC_CODE)` returns true for `PT_COUNT_STAR`, `PT_COUNT`, `PT_MIN`, `PT_MAX`, `PT_SUM`, `PT_AVG`, `PT_STDDEV`, `PT_STDDEV_POP`, `PT_STDDEV_SAMP`, `PT_VARIANCE`, `PT_VAR_POP`, `PT_VAR_SAMP`. Replaces the inline `agg_it->function != PT_COUNT_STAR && agg_it->function != PT_COUNT` check at the BUILDVALUE_PROC arm.
- **Local variable rename**: `count_opt` → `buildvalue_opt`; `CANNOT_COUNT_OPT` flag → `CANNOT_BUILDVALUE_OPT`.
- **Trace label rename** in `px_heap_scan_trace_handler.cpp`: `result_type_str = ... ? "count" : "unknown"` → `... ? "buildvalue" : "unknown"`. Affects `text` and `json` trace output formats both.
- **Template instantiations** in 4 files (`px_heap_scan.cpp`, `px_heap_scan_result_handler.cpp`, `px_heap_scan_task.cpp`, `query_executor.c`): all `manager<COUNT_DISTINCT>` / `task<COUNT_DISTINCT>` / `result_handler<COUNT_DISTINCT>` template specializations renamed in the `extern "C"` switch arms. Explicit instantiations at file end (`template class manager<RESULT_TYPE::BUILDVALUE_OPT>;` etc.) likewise.
- **Thread-local storage instances** (`px_heap_scan_result_handler.cpp` lines 47-52): `result_handler<RESULT_TYPE::COUNT_DISTINCT>::tl_agg_p`, `tl_outptr_list_p`, `tl_vd`, `tl_xasl_p`, `tl_tpl_buf`, `tl_or_buf` all renamed to the `BUILDVALUE_OPT` specialization.
- **No new files. No new public symbols. ABI-internal change** — `RESULT_TYPE` enum is process-local; `ACCESS_SPEC_FLAG` is client-side XASL annotation, not serialized.

### Per-file notes

- `src/query/parallel/px_heap_scan/px_heap_scan.cpp` — pure renames in 5 switch arms (open/start/next/reset/end/close) + 2 explicit instantiations + the `result_type` selection block (`ACCESS_SPEC_IS_FLAGED → ACCESS_SPEC_FLAG_BUILDVALUE_OPT`). Documented in [[components/parallel-heap-scan]].
- `src/query/parallel/px_heap_scan/px_heap_scan_checker.cpp` — adds `is_buildvalue_opt_supported_function` helper + renames `count_opt`/`CANNOT_COUNT_OPT` everywhere. The check at line ~512 (BUILDVALUE_PROC arm) now uses the helper instead of a hardcoded 2-function whitelist. Documented in [[components/parallel-heap-scan-support]].
- `src/query/parallel/px_heap_scan/px_heap_scan_result_handler.cpp` — **the main engineering work**. Six method bodies rewritten for the `BUILDVALUE_OPT` specialization: `read_initialize`, `read`, `write_initialize`, `write`, `write_finalize`, plus the constructor (renamed). Detailed walkthrough below. Documented in [[components/parallel-heap-scan-result-handler]].
- `src/query/parallel/px_heap_scan/px_heap_scan_result_handler.hpp` — class-template specialization name rename.
- `src/query/parallel/px_heap_scan/px_heap_scan_result_type.hpp` — enum value rename + comment.
- `src/query/parallel/px_heap_scan/px_heap_scan_task.cpp` — renames in 6 `if constexpr` branches; **adds 4-line error propagation** after `write_initialize` (`if (er_errid () != NO_ERROR) return er_errid ();`) — captures the new failure modes added in `write_initialize` (alloc failures now set ER_OUT_OF_VIRTUAL_MEMORY + interrupt code). Documented in [[components/parallel-heap-scan-task]].
- `src/query/parallel/px_heap_scan/px_heap_scan_trace_handler.cpp` — trace label "count" → "buildvalue" in two places (text + json variants).
- `src/query/query_executor.c` — single switch arm rename in `qexec_clear_access_spec_list`. Documented in [[components/query-executor]].
- `src/query/xasl.h` — single enum bit rename (preserved value `0x1 << 4`).

### Behavioral

The substantive behavior changes all live in `px_heap_scan_result_handler.cpp`:

**`write_initialize`** — per-worker setup:
- Forces `tl_xasl_p->proc.buildvalue.agg_domains_resolved = 0` (new line) so each worker's domain resolution happens fresh on its first qualified row. Previously, domain resolution state from a prior scan could leak.
- Three-way branch by aggregate kind:
  1. `PT_COUNT_STAR` → `acc.curr_cnt = 0`
  2. `Q_DISTINCT` (excluding MIN/MAX) → opens per-thread `QFILE_LIST_ID` for the operand domain (existing pattern); **NEW**: on `db_private_alloc` failure for `type_list.domp` OR `qfile_open_list` failure, calls `m_err_messages_p->move_top_error_message_to_this()` and sets `interrupt_code::ERROR_INTERRUPTED_FROM_WORKER_THREAD`. Previously the failures returned silently; now they propagate.
  3. Everything else (`PT_COUNT`, `PT_MIN`, `PT_MAX`, `PT_SUM`, `PT_AVG`, `PT_STDDEV*`, `PT_VAR*`) → `acc.curr_cnt = 0`. The `value`/`value2` fields stay untouched here; first-row coercion happens lazily in `write()`.

**`write`** — per-row aggregation, now a multi-way switch:
- Refactored common path: a single `fetch_peek_dbval` (or `TYPE_CONSTANT` shortcut) at the top extracts the first operand's `db_value_p`. Previously the code re-fetched per branch.
- `DB_IS_NULL(db_value_p)` → continue (NULL-skipping per SQL aggregate semantics).
- `Q_DISTINCT` (excluding MIN/MAX) → re-fetches per operand (because there can be multiple, e.g. `COUNT(DISTINCT a, b)`), writes each operand's value to the per-thread `QFILE_LIST_ID` for de-duplication during the merge phase.
- Otherwise switches on `agg_node->function`:
  - `PT_COUNT` → `acc->curr_cnt++`
  - `PT_MIN` / `PT_MAX` → if `curr_cnt < 1` OR cmpval shows new value is smaller/larger (using `acc_dom->value_dom->collation_id`): `pr_clear_value(acc->value)`, then either `db_value_coerce` (if domain mismatch) or `pr_clone_value`, then `acc->curr_cnt++`. Collation-aware comparison.
  - `PT_SUM` / `PT_AVG` → first row clones (with optional coerce); subsequent rows call `qdata_add_dbval(acc->value, db_value_p, acc->value, acc_dom->value_dom)` to incrementally sum.
  - `PT_STDDEV*` / `PT_VAR*` → `tp_value_coerce` to accumulator domain, `qdata_multiply_dbval` to compute X², then accumulate sum-of-X in `acc->value` and sum-of-X² in `acc->value2`. First row uses `setval(..., true)` (deep clone); subsequent rows `qdata_add_dbval` both halves. Cleans up `coerced` and `squared` on every error path.
  - default → `assert(false)` (compile-time guarantee that the checker filters anything else).

**`write_finalize`** — per-worker partial → shared accumulator merge, under `writer_results_mutex`:
- **NEW**: at the top of each iteration, checks `m_interrupt_p->get_code() != NO_INTERRUPT`; if interrupted, walks the rest of the agg list freeing list_ids (DISTINCT case) and clearing accumulator values, then breaks. Prevents leak/double-free on cancellation.
- `PT_COUNT_STAR` → `orig.curr_cnt += worker.curr_cnt` (unchanged).
- `Q_DISTINCT` (non-MIN/MAX) → existing `qfile_connect_list` flow, but **the `malloc(QFILE_LIST_ID)` now has explicit failure handling**: sets `ER_OUT_OF_VIRTUAL_MEMORY`, calls `move_top_error_message_to_this`, sets interrupt code, destroys the worker's list, advances iterator, continues. (Greptile P1 fix.)
- `PT_COUNT` (non-DISTINCT) → `orig.curr_cnt += worker.curr_cnt` (unchanged structurally; moved out of the previous nested `else` branch).
- Everything else (MIN/MAX/SUM/AVG/STDDEV*/VAR*) → **the new accumulator-merge fast path**:
  1. If `orig.accumulator_domain.value_dom == NULL && worker.accumulator_domain.value_dom != NULL`, copy worker's resolved domain pointers into orig. (Worker resolved the domain on first row; main may not have seen any row.)
  2. Switch private heap to **heap 0** via `db_change_private_heap(thread_p, 0)`.
  3. Call `qdata_aggregate_accumulator_to_accumulator(thread_p, &orig.acc, &orig.acc_dom, function, domain, &worker.acc)` — the standard CUBRID accumulator merge primitive.
  4. Restore previous heap.
  5. On error: `move_top_error_message_to_this` + interrupt set.

**`read_initialize`** — main-thread setup before consumption:
- Previously: unconditionally set `orig_agg_p->accumulator_domain.value_dom = &tp_Bigint_domain` and `value2_dom = &tp_Null_domain` for every aggregate (correct only for COUNT). **Now**: only does that for `PT_COUNT_STAR` and `PT_COUNT`. Other aggregates keep their resolved domains (from XASL semantic check or first-row resolution by workers).

**`read`** — main thread post-merge cleanup, the **two-heap dance**:
- `PT_COUNT_STAR` → no-op (curr_cnt already merged in finalize).
- `Q_DISTINCT` (non-MIN/MAX) → no-op here; final list cardinality counted later by `qexec_end_buildvalueblock_iterations`.
- `PT_COUNT` → `db_make_bigint(acc.value, curr_cnt)` (unchanged).
- Everything else → **re-clone accumulator values from heap 0 into the calling thread's private heap**:
  1. If `acc->value` non-NULL: `pr_clone_value(acc->value, &tmp)` (clones into main thread's private heap by default).
  2. Switch to heap 0 via `db_change_private_heap`.
  3. `pr_clear_value(acc->value)` — frees the heap-0 allocation that was made by workers.
  4. Restore previous heap.
  5. Assign cloned tmp back: `*acc->value = tmp`.
  6. Same dance for `acc->value2` (used by STDDEV/VAR).
- **Why**: `qexec_end_buildvalueblock_iterations` (downstream cleanup) calls `pr_clear_value` on the accumulators expecting them to be in the calling thread's private heap. Worker writes made values in heap 0 (so they survive worker teardown); main needs to relocate them before the standard cleanup runs. Without this dance, the cleanup would try to free heap-0 memory through the wrong heap and crash.

### Performance characteristics

- **Worker-side cost per row**: O(1) for COUNT/SUM/AVG (one DB_VALUE add), O(1) for MIN/MAX (one cmpval + conditional clone), O(1) for STDDEV/VAR (one square + two adds). Compared to the old fallback (MERGEABLE_LIST), this avoids per-tuple list_file write + tuple serialization.
- **Merge cost in main thread**: O(parallelism × num_aggregates) — one `qdata_aggregate_accumulator_to_accumulator` call per (worker, agg) pair. Independent of row count.
- **Total expected speedup vs MERGEABLE_LIST fallback**: substantial for large tables — was O(rows × per-tuple-serialize-cost) materialization, now O(rows / parallelism × per-row-add-cost) with O(parallelism) finalization.

### New surface (no existing wiki reference)

- `is_buildvalue_opt_supported_function` (file-static helper in `px_heap_scan_checker.cpp`) — minor enough to mention inline in [[components/parallel-heap-scan-support]] rather than its own page.
- The two-heap dance (heap-0 worker writes ↔ main-thread re-clone) is a new pattern in this codebase area. Documented under "Cross-thread DB_VALUE ownership" in the reconciled [[components/parallel-heap-scan-result-handler]] page.

## Review discussion highlights

- **greptile-apps[bot]** filed 14 of the 17 inline comments, mostly P1 (crash-risk) about missing NULL checks on `db_private_alloc`/`db_private_realloc`/`malloc`, missing error propagation from `write_initialize`, missing cleanup in error paths. The author addressed each via a sequence of 6 `greptile review apply` commits (b70d18b, f2e6c00, 0893ec0, 593d347, 018aaef, 9139da9). Final state: every alloc has a NULL check, every error path frees its locals, `write_initialize` propagates failures via `move_top_error_message_to_this()` + interrupt code (consumed by the new check in `task.cpp` after `write_initialize`).
- **greptile P1 — `MIN(DISTINCT)` / `MAX(DISTINCT)` list_id resource leak**: opening the per-thread DISTINCT list for MIN/MAX was wasteful because MIN/MAX(DISTINCT) is semantically identical to MIN/MAX (no distinctness affects extrema). Author fix (commit `0671e35` "err handling, min max distinct is no useful"): excluded MIN/MAX from the DISTINCT path entirely (`agg_node->option == Q_DISTINCT && agg_node->function != PT_MIN && agg_node->function != PT_MAX`). All five branches (write_initialize, write, write_finalize, read) carry the same MIN/MAX-distinct exclusion guard.
- **greptile P1 — `pr_clone_value` failure → silent aggregate corruption**: the original drafts ignored the `int` return of `pr_clone_value`. Fixed in `593d347` to check return + propagate via `return false` in `write` and via interrupt-set in `write_finalize`.
- **xmilex-git note on `er_errid()` check** (line 1056): in response to greptile P1 about `write_initialize` not propagating errors, author wrote "er_errid()로 체크했어" — refers to the new 4-line check added in `px_heap_scan_task.cpp` immediately after `write_initialize` (returns `er_errid()` if non-zero). Confirms the design choice: failures inside `write_initialize` set the thread-local error via `er_set`, and the caller polls `er_errid()` to decide whether to continue.
- **Final commit `c047445c`** "Apply review feedback: reuse first-operand fetch in DISTINCT loop" — addresses a minor redundancy where the DISTINCT-loop body re-fetched `db_value_p` per operand even when `operand == agg_node->operands` (the same operand the outer block already fetched). Optimization only; semantics unchanged.

## Reconciliation Plan

Promoted to **Pages Reconciled** below — applied during this ingest.

## Pages Reconciled

- [[components/parallel-heap-scan]] — RESULT_TYPE table updated: `COUNT_DISTINCT` row renamed to `BUILDVALUE_OPT`, description widened from "for UPDATE STATISTICS" / "aggregate-only" to "BUILDVALUE_PROC fast path: COUNT/MIN/MAX/SUM/AVG/STDDEV/VAR partial-aggregate per worker". Result-mode-selection block likewise updated. `> [!update] PR #7049 (65d6915)` callout added.
- [[components/parallel-heap-scan-result-handler]] — major update:
  - RESULT_TYPE Taxonomy table: `COUNT_DISTINCT` row renamed to `BUILDVALUE_OPT`; column descriptions updated.
  - "Full specialisation `result_handler<COUNT_DISTINCT>`" section heading + table renamed to `BUILDVALUE_OPT`; method descriptions updated to reflect the new write-time aggregation switch + finalize merge protocol.
  - "Execution Path — COUNT_DISTINCT" subsection rewritten as "Execution Path — BUILDVALUE_OPT" with the new write/write_finalize/read flow including the heap-0 ↔ private-heap dance.
  - **New subsection added**: "Cross-thread DB_VALUE ownership (heap 0 ↔ private heap)" documenting the two-heap dance and why it's needed (lifetime of accumulator vs. worker thread teardown vs. caller's `pr_clear_value` expectations).
  - `> [!update] PR #7049 (65d6915)` callout at top.
- [[components/parallel-heap-scan-task]] — RESULT_TYPE references updated; `> [!update]` callout noting the new 4-line `er_errid()` check after `write_initialize` and the `BUILDVALUE_OPT` rename throughout the `if constexpr` branches.
- [[components/parallel-heap-scan-support]] — checker section: `is_buildvalue_opt_supported_function` helper documented inline (12-aggregate whitelist); `count_opt` → `buildvalue_opt` rename; `CANNOT_COUNT_OPT` → `CANNOT_BUILDVALUE_OPT`. `> [!update]` callout.
- [[components/aggregate-analytic]] — added section "Parallel-heap-scan fast path" listing the 12 aggregates that can now run via BUILDVALUE_OPT in parallel (was 2 before this PR). Cross-link to [[components/parallel-heap-scan-result-handler]].
- [[components/xasl-aggregate]] — note: `ACCESS_SPEC_FLAG_BUILDVALUE_OPT` (renamed from `_COUNT_DISTINCT`, bit 4) now gates a much wider class of aggregates.
- [[components/query-executor]] — `qexec_clear_access_spec_list` switch arm renamed; trivial rename mention.

## Incidental wiki enhancements

- [[components/aggregate-analytic]] — added explicit cross-reference to the BUILDVALUE_OPT fast path (was previously not linked from the aggregate page).
- [[components/parallel-heap-scan-result-handler]] — documented `qdata_aggregate_accumulator_to_accumulator` as the standard accumulator-merge primitive (baseline truth that wasn't called out before — applies regardless of this PR's broader rollout).

## Baseline impact

- Before: `175442fc858bd0075165729756745be6f8928036`
- After: `65d6915437eb6217ab0050939c6ad63f0d509735`
- Bump triggered: `true`
- Logged: [[log]] under `## [2026-04-27] baseline-bump | 175442fc → 65d69154 (PR #7049)`

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: <https://github.com/CUBRID/cubrid/pull/7049>
- Jira: [CBRD-26711](https://jira.cubrid.org/browse/CBRD-26711)
- Components: [[components/parallel-heap-scan]], [[components/parallel-heap-scan-result-handler]], [[components/parallel-heap-scan-task]], [[components/parallel-heap-scan-support]], [[components/aggregate-analytic]], [[components/xasl-aggregate]], [[components/query-executor]]
- Sources: [[sources/cubrid-src-query-parallel]], [[sources/cubrid-src-query]], [[sources/cubrid-src-xasl]]
- Sibling PR (related ground): [[prs/PR-7011-parallel-index-build]] — also touched the parallel-heap-scan family this week (`px_ftab_set` move, declared-but-unused `SORT_INDEX_LEAF` wired up).
