---
type: pr
pr_number: 7011
pr_url: "https://github.com/CUBRID/cubrid/pull/7011"
repo: "CUBRID/cubrid"
state: MERGED
is_draft: false
author: "xmilex-git"
created_at:
merged_at: "2026-04-27T05:20:37Z"
closed_at: "2026-04-27T05:20:37Z"
merge_commit: "cc563c7fd90521393781d8440bf5144d2566ff71"
base_ref: "develop"
head_ref: "parallel_index_build"
base_sha: "175442fc858bd0075165729756745be6f8928036"
head_sha: "6f5ca7ae2ad2bc2f49962ee86c0e28470e77eb0a"
jira: "CBRD-26678"
files_changed:
  - "cubrid/CMakeLists.txt"
  - "src/query/parallel/px_heap_scan/px_heap_scan_ftab_set.hpp → src/query/parallel/px_ftab_set.hpp (renamed)"
  - "src/query/parallel/px_heap_scan/px_heap_scan_input_handler_ftabs.hpp"
  - "src/storage/btree_load.c (+468/-46)"
  - "src/storage/btree_load.h (+73/-0)"
  - "src/storage/external_sort.c (+425/-46)"
  - "src/storage/external_sort.h (+2/-0)"
  - "src/storage/file_manager.c (+25/-0)"
  - "src/storage/file_manager.h (+1/-0)"
related_components:
  - "[[components/btree]]"
  - "[[components/external-sort]]"
  - "[[components/parallel-sort]]"
  - "[[components/parallel-heap-scan-input-handler]]"
  - "[[components/parallel-heap-scan]]"
  - "[[components/parallel-query]]"
  - "[[components/file-manager]]"
  - "[[components/storage]]"
related_sources:
  - "[[sources/cubrid-src-storage]]"
  - "[[sources/cubrid-src-query-parallel]]"
ingest_case: c
triggered_baseline_bump: true
baseline_before: "65d6915437eb6217ab0050939c6ad63f0d509735"
baseline_after: "cc563c7fd90521393781d8440bf5144d2566ff71"
reconciliation_applied: true
reconciliation_applied_at: 2026-04-27
incidental_enhancements_count: 1
tags:
  - pr
  - cubrid
  - parallel-query
  - btree
  - sort
  - index-build
  - merged
created: 2026-04-26
updated: 2026-04-27
status: merged
---

# PR #7011 — Support parallel index build

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `MERGED` · **Author:** `@xmilex-git` · **Jira:** [CBRD-26678](https://jira.cubrid.org/browse/CBRD-26678)
> **Merged:** 2026-04-27T05:20:37Z · **Merge commit:** `cc563c7fd90521393781d8440bf5144d2566ff71`
> **Base → Head:** `develop` (`175442fc8`) → `parallel_index_build` (`6f5ca7ae2` final, was `44d92db64` at first ingest)
> **Scale:** 9 files, +1058/−99 (1157 LOC). Largest: `btree_load.c` (+468/−46 = 514 changed), `external_sort.c` (+425/−46 = 471 changed). 1 file > 500 LOC. Original analysis via 2 parallel deep-read subagents (2026-04-26 OPEN ingest).
> **Approvals at merge:** @hornetmj (2026-04-20), @shparkcubrid (2026-04-23), @youngjinj (2026-04-23), @sohee-dgist (2026-04-24), @Hamkua (2026-04-24). All 5 carried through to merge unchanged.

> [!note] Ingest classification: `c` (newer than baseline)
> Reconciled on 2026-04-27. Originally ingested OPEN on 2026-04-26 against baseline `175442fc`; baseline subsequently bumped to `65d69154` via [[prs/PR-7049-parallel-buildvalue-heap|PR #7049]]. PR #7011 merged as commit `cc563c7f` (direct child of `65d69154`) on 2026-04-27 at 05:20Z. Reconciliation Plan promoted to "Pages Reconciled". Baseline bumped `65d6915` → `cc563c7`. Between snapshot head `44d92db` and final head `6f5ca7a` only one new commit landed: `6f5ca7a` (`Merge branch 'CUBRID:develop' into parallel_index_build`) — no logic changes to PR-touched files (verified via `git diff` on the 9 touched files). Original code analysis remains accurate.

## Summary

Parallelizes the heap-scan + sort phase of `xbtree_load_index` (CREATE INDEX) by reusing the existing parallel-sort infrastructure (`SORT_EXECUTE_PARALLEL` / `SORT_WAIT_PARALLEL`). Adds wiring for the previously-declared-but-unused `SORT_INDEX_LEAF` `SORT_PARALLEL_TYPE` enum value. Heap is partitioned by sectors via `file_get_all_data_sectors` + `ftab_set::split(parallel_num)`; each worker scans its assigned sector subset (`btree_sort_get_next_parallel` + `get_next_vpid`), produces a thread-local sorted run in a temp file, then a log₄ tree-merge consolidates the per-worker runs into a final btree leaf via `sort_merge_run_for_parallel_index_leaf_build`. SERVER_MODE delegates `log_sysop_start` / `btree_create_file` / `vacuum_log_add_dropped_file` lifecycle to the sort layer; SA_MODE retains the existing serial path with btree_load.c-side setup. The class `parallel_heap_scan::ftab_set` is generalized to `parallel_query::ftab_set` (file moved out of `px_heap_scan/`) so both heap-scan and index-build can share it.

## Motivation

CREATE INDEX on large tables is bottlenecked by the heap-scan + sort step. The existing parallel-sort infrastructure already supports ORDER BY parallelism via per-worker run generation + tree-merge; this PR reuses that infrastructure for the index-build case. CBRD-26678 targets noticeable wall-time reduction on multi-million-row tables under modest parallelism (default `parallel_num` is small single digits via `compute_parallel_degree`).

## Changes

### Structural

#### Renamed file (with C++ namespace migration)

| Before | After |
|---|---|
| `src/query/parallel/px_heap_scan/px_heap_scan_ftab_set.hpp` | `src/query/parallel/px_ftab_set.hpp` |
| `namespace parallel_heap_scan { class ftab_set { ... } }` | `namespace parallel_query { class ftab_set { ... } }` |

`parallel_heap_scan::ftab_set` is preserved as a `using`-alias inside `px_heap_scan_input_handler_ftabs.hpp`:

```cpp
namespace parallel_heap_scan {
  using ftab_set = parallel_query::ftab_set;
  ...
}
```

This keeps existing heap-scan call-sites compiling unchanged while letting `external_sort.c` and `btree_load.c` consume the same class under its new home namespace. CMakeLists moves the header from `PARALLEL_HEAP_SCAN_HEADERS` to `PARALLEL_QUERY_HEADERS`.

#### `parallel_query::ftab_set` extended

The class gains:
- Destructor `~ftab_set() { m_ftab_set.clear(); }`
- Copy and move constructors
- Copy and move assignment operators
- `void append(const ftab_set &other)` — concatenates `m_ftab_set` lists
- `void move_from(ftab_set &other)` — `std::move` content + reset other's iterator
- `size_t size() const` — exposes the underlying vector size

These additions support the new "split into per-worker `ftab_set`s and pass by value/move into per-worker `SORT_ARGS`" pattern.

#### `SORT_ARGS` promoted from file-static to public header

`SORT_ARGS` was previously a `static` typedef inside `btree_load.c` (~lines 57-86). The PR moves the full struct definition to `btree_load.h` and extends it. This is necessary because `external_sort.c` (`sort_listfile_execute`, `sort_start_parallelism`, `sort_merge_run_for_parallel_index_leaf_build`, `sort_return_used_resources`, `sort_check_parallelism`) now dereferences `sort_param->get_arg` as `SORT_ARGS *`.

**New `SORT_ARGS` fields:**

| Field | Type | Role |
|---|---|---|
| `filter_index_info` | `FILTER_INDEX_INFO *` | Carries the raw `pred_stream` / `pred_stream_size` so each parallel worker can re-deserialize the XASL filter predicate from the byte stream (XASL trees are not thread-safe). Set under `SERVER_MODE` from `xbtree_load_index` before parallel split. |
| `ftab_sets` | `std::vector<parallel_query::ftab_set> *` | Per-thread copy of the file-table-sector partition: index `[cur_class]` returns the `ftab_set` for the current heap class. Owned by per-thread `SORT_ARGS`. Allocated via `malloc` + placement-`new`; freed via explicit `~vector()` + `free_and_init`. |
| `curr_sec` | `FILE_PARTIAL_SECTOR` | Iteration cursor: the partial sector currently being walked by `get_next_vpid`. NULL-initialized; reset to NULL via `VSID_SET_NULL` when sector is exhausted. |
| `curr_pgoffset` | `int` | Iteration cursor: page offset within `curr_sec` (0..`DISK_SECTOR_NPAGES-1`). |

Notably **no `n_thd` / `worker_thread_idx` field** — workers are not numerically self-aware; their state is fully captured by `ftab_sets`/`curr_sec`/`curr_pgoffset`.

#### New public type `FILTER_INDEX_INFO`

In `btree_load.h`:

```c
typedef struct filter_index_info FILTER_INDEX_INFO;
struct filter_index_info {
  char *pred_stream;
  int   pred_stream_size;
};
```

Mirrors the pre-existing `FUNCTION_INDEX_INFO` but carries only the raw byte stream (no unpacked `expr` pointer) — by design, since each worker re-unpacks into its own `PRED_EXPR_WITH_CONTEXT`.

#### Newly extern'd functions (promoted from `static` to public header)

| Function | Pre-PR linkage | Post-PR | New caller |
|---|---|---|---|
| `bt_load_heap_scancache_start_for_attrinfo` | `static` | `extern` | `external_sort.c::sort_listfile`, `sort_listfile_execute` |
| `bt_load_heap_scancache_end_for_attrinfo` | `static` | `extern` | same |
| `bt_load_clear_pred_and_unpack` | `static` | `extern` | `external_sort.c::sort_listfile_execute` |
| `btree_load_filter_pred_function_info` | did not exist | `extern` (new) | `external_sort.c::sort_listfile_execute` |
| `btree_sort_get_next_parallel` | did not exist | `extern` (new) | `external_sort.c::sort_start_parallelism` (assigned as `get_fn`) |

`btree_sort_get_next_parallel` is declared in `external_sort.h:165` rather than `btree_load.h` — minor cross-module smell (see Code concerns §3 below).

#### New file_manager helper

```c
int file_get_num_data_sectors (THREAD_ENTRY *thread_p, const VFID *vfid, int *n_sectors_out);
```

Reads `*n_sectors_out = fhead->n_sector_full + fhead->n_sector_partial;` — direct assignment, not accumulation. Called by `sort_check_parallelism` to decide whether parallelism is worthwhile (compare sector count to threshold).

#### New constants
- `SORT_PX_MERGE_FILES = 4` (already in `px_sort.h:36`) — fan-in degree for the tree merge in `sort_merge_run_for_parallel_index_leaf_build`.
- `SORT_MAX_PARALLEL = PRM_MAX_PARALLELISM` (`px_sort.h:37`).

### Per-file notes

- `cubrid/CMakeLists.txt` (+1/-1) — moves `px_heap_scan_ftab_set.hpp` listing from `PARALLEL_HEAP_SCAN_HEADERS` to `PARALLEL_QUERY_HEADERS` (and renames to `px_ftab_set.hpp`).
- `src/query/parallel/px_ftab_set.hpp` (renamed, +56/-5) — namespace change `parallel_heap_scan` → `parallel_query`; gains move/copy semantics + helpers ([[components/parallel-query]] / [[components/parallel-heap-scan-input-handler]]).
- `src/query/parallel/px_heap_scan/px_heap_scan_input_handler_ftabs.hpp` (+2/-1) — adds `using ftab_set = parallel_query::ftab_set;` alias to keep existing heap-scan code compiling ([[components/parallel-heap-scan-input-handler]]).
- `src/storage/btree_load.c` (+468/-46) — `SORT_ARGS` extracted from this file; new helpers `btree_sort_get_next_parallel`, `get_next_vpid`, `btree_load_filter_pred_function_info`; existing `bt_load_heap_scancache_*` exported; `log_sysop_start` shifted out for SERVER_MODE ([[components/btree]]).
- `src/storage/btree_load.h` (+73/-0) — gains `SORT_ARGS`, `FILTER_INDEX_INFO`, `parallel_query::ftab_set` forward decl, 4 new extern'd prototypes ([[components/btree]]).
- `src/storage/external_sort.c` (+425/-46) — flagship parallel-merge function `sort_merge_run_for_parallel_index_leaf_build`; `SORT_INDEX_LEAF` arms in `sort_listfile`, `sort_listfile_execute`, `sort_return_used_resources`, `sort_check_parallelism`, `sort_start_parallelism`, `sort_end_parallelism`. `sort_copy_sort_param` newly **implemented** (resolves a previously-flagged baseline gap) ([[components/external-sort]] / [[components/parallel-sort]]).
- `src/storage/external_sort.h` (+2/-0) — new extern decl for `btree_sort_get_next_parallel` ([[components/external-sort]]).
- `src/storage/file_manager.c` (+25/-0) — adds `file_get_num_data_sectors` ([[components/file-manager]]).
- `src/storage/file_manager.h` (+1/-0) — prototype ([[components/file-manager]]).

### Behavioral

1. **Parallel index-build dispatch.** `xbtree_load_index` → `btree_index_sort` → `sort_listfile` (SERVER_MODE). If `sort_check_parallelism` returns N > 1 (sector count exceeds threshold and `n_classes == 1`), parallelism kicks in; otherwise single-process path runs.
2. **Per-worker setup** in `sort_start_parallelism` (single pass, post-fix `e71b21b`):
   ```c
   for (int i = 0; i < parallel_num; i++) {
       px_sort_args_p = malloc(sizeof(SORT_ARGS));
       memcpy(px_sort_args_p, sort_param->get_arg, sizeof(SORT_ARGS));
       px_sort_param[i].get_arg = px_sort_args_p;
       px_sort_param[i].get_fn  = &btree_sort_get_next_parallel;  // override serial getter
       px_sort_args_p->ftab_sets    = NULL;
       px_sort_args_p->curr_sec     = FILE_PARTIAL_SECTOR_INITIALIZER;
       px_sort_args_p->curr_pgoffset = 0;
       px_sort_args_p->in_recdes    = { 0, 0, 0, NULL };
   }
   ```
   Then `file_get_all_data_sectors` collects all heap sectors per class, `temp.split(parallel_num)` partitions, and per-worker `ftab_sets` vectors are populated (one entry per `n_classes`).
3. **Sector-walk iteration** (`get_next_vpid` in btree_load.c). Per-worker `pgbuf_ordered_fix` of pages within assigned sectors; `pgbuf_replace_watcher` rotates pin between consecutive pages; `pgbuf_ordered_unfix(old_pgwatcher)` releases the previous page on advance. On `ER_PB_BAD_PAGEID` (page deallocated by concurrent vacuum/DROP between snapshot and worker access), `er_clear()` + `continue` — benign race since the sector bitmap was sampled at scan start. **Bug fix `ab8ca3a`**: pre-fix, the two early-return paths (sector exhausted → `S_END`, fix error → `S_ERROR`) returned without unfixing `old_pgwatcher`, leaking the previous page's pin. Now both paths call `pgbuf_ordered_unfix` if `old_pgwatcher->pgptr != NULL`.
4. **Per-worker scancache lifecycle.** Each worker calls `bt_load_heap_scancache_start_for_attrinfo` at the top of `sort_listfile_execute` (after a `memset` of the local `hfscan_cache`/`attr_info`), runs the sort, then `bt_load_heap_scancache_end_for_attrinfo` at cleanup. Workers inherit `tran_index` from `sort_param->px_orig_thread_p`, so `db_private_alloc` allocations are tran-scoped (not thread-local).
5. **Per-worker XASL re-deserialization.** Each worker takes a stack-local `FILTER_INDEX_INFO` snapshot of the parent's `pred_stream` pointers, NULLs its own copy of `sort_args_p->filter` and `func_index_info`, then calls `btree_load_filter_pred_function_info` to re-unpack:
   - `stx_map_stream_to_filter_pred(pred_stream)` → fresh `PRED_EXPR_WITH_CONTEXT` per worker.
   - `stx_map_stream_to_func_pred(expr_stream)` → fresh `FUNC_PRED::expr` per worker plus its own `XASL_UNPACK_INFO` arena.
   - `eval_fnc(...)` recomputes the `filter_eval_func` for the freshly unpacked predicate.
   `bt_load_clear_pred_and_unpack` releases everything in cleanup.
   **Why per-worker:** `PRED_EXPR_WITH_CONTEXT::cache_pred` (a `HEAP_CACHE_ATTRINFO`) is mutated during evaluation. Sharing one unpacked predicate across N workers would race on this cache. Memory cost: O(parallel_num) × tree-size, typically a few KB per worker.
6. **Tree-merge fan-in** in `sort_merge_run_for_parallel_index_leaf_build`. SORT_PX_MERGE_FILES = 4, depth = `ceil(log₄ parallel_num)`. For `parallel_num = 8`: 2 levels (8 → 2 → 1). For 64: 3 levels. Empty (0-page) workers are skipped during input gathering — only non-empty runs are passed into a merge slot. If `valid_j ≤ 1` for a slot, no merge dispatched (lone non-empty run propagated directly). After all merges complete, parent `SORT_ARGS` accumulates `n_oids` and `n_nulls` from per-worker copies.
7. **Sysop & btree_create_file location** (corrected by commit `42b92007` "Set is_sysop_started under SERVER_MODE guard after sort"):
   - SA_MODE: `log_sysop_start` at top of `xbtree_load_index` (`btree_load.c:906`); `btree_create_file` follows immediately. Existing serial flow.
   - SERVER_MODE single-process: `log_sysop_start` + `btree_create_file` inside `sort_listfile` at `external_sort.c:1500-1501`, just before `sort_listfile_internal`.
   - SERVER_MODE parallel: `log_sysop_start` + `btree_create_file` + `vacuum_log_add_dropped_file` inside `sort_merge_run_for_parallel_index_leaf_build` at `external_sort.c:4848-4858`, **after** the per-thread sorts complete and the merge fan-in has consolidated runs.
   In all SERVER_MODE paths the sysop is left **open on success** for the caller (`xbtree_load_index`) to close — the caller sets `is_sysop_started = true` (under `#if defined(SERVER_MODE)`) at `btree_load.c:1085` and aborts/commits via the function-level `error:` label. On failure, the sort layer always calls `log_sysop_abort` before returning ER_FAILED.
8. **Why `btree_create_file` was hoisted into the merge phase** (parallel path only): each worker sorts into its own thread-local temp file, not into the final btree. Creating the btree file before workers start would (a) leave a half-built btree visible if any worker errored and forced the whole sysop to abort, and (b) require the sysop to span the entire heap-scan window — a long latch-holding period. Deferring shortens the sysop's life and means worker-fail never produces a partial btree file.
9. **`SORT_INDEX_LEAF` short-circuits multi-class parallelism.** `sort_check_parallelism` returns 1 (single-process) if `n_classes > 1` — multi-class indexes (inheritance / partition-base) are not parallelized in this PR. The plumbing supports it (per-worker `ftab_sets` is `vector` indexed by `cur_class`, with empty entries pushed when classes are skipped) but the gate is conservative. Future-proofed.
10. **Resolves baseline gap**: the wiki's [[hot|hot cache]] flagged "`sort_copy_sort_param` declared in `px_sort.h` but implementation missing in `px_sort.c`". This PR provides the implementation in `external_sort.c:4344-4471` (memcpy of `SORT_PARAM` per-worker, then null aliased pointers, then re-allocate per-worker resources).
11. **Silent fixes** the PR drops in (not called out in the body):
    - `external_sort.c:1491` — `if (sort_param == NULL)` → `if (px_sort_param == NULL)` after the worker-array `malloc` (was always-true on the wrong pointer).
    - `external_sort.c:1102` — `malloc(...)` → `calloc(...)` for `file_contents[j].num_pages`. Avoids a possible uninit read for empty-worker runs in the tree merge.

### New surface (no existing wiki reference)

- `parallel_query::ftab_set` namespace + extended class — `[[components/parallel-heap-scan-input-handler]]` currently documents the `parallel_heap_scan::ftab_set` form.
- `SORT_INDEX_LEAF` activation — currently no wiki page documents the SORT_PARALLEL_TYPE enum's full surface.
- `btree_sort_get_next_parallel` + `get_next_vpid` + `btree_load_filter_pred_function_info` + `bt_load_clear_pred_and_unpack` — no wiki page covers `btree_load.c`'s per-thread getter family.
- `sort_merge_run_for_parallel_index_leaf_build` — `[[components/parallel-sort]]` covers the ORDER_BY tree merge but not the index-leaf variant.
- `file_get_num_data_sectors` — no [[components/file-manager]] reference.

## Review discussion highlights

5 approvals across 4 reviewers. Greptile bot flagged 4 P0/P1 + several P2 issues, all addressed at PR head. Significant inline threads:

- **`@greptile-apps[bot] @ external_sort.c:4857` (P0)** — `log_sysop_start` cleanup gap on `btree_create_file` / `sort_put_result_from_tmpfile` failure. **Addressed**: both error returns now call `log_sysop_abort`. Success path leaves sysop open for caller (xbtree_load_index) to close — a callee-leaves-sysop-open contract worth documenting.
- **`@greptile-apps[bot] @ external_sort.c:4251` (P1)** — `new`-allocated `ftab_sets` freed via `free_and_init` (UB). **Addressed**: now uses `malloc` + placement-`new` paired with explicit `~vector()` + `free_and_init` in `sort_return_used_resources`.
- **`@greptile-apps[bot] @ external_sort.c:5278` (P1)** — `SORT_ARGS` allocation failure double-free hazard. **Addressed**: `sort_start_parallelism` SORT_INDEX_LEAF arm pre-NULLs every `px_sort_param[i].get_arg` before the malloc loop; `sort_return_used_resources` checks `if (sort_param->get_arg != NULL)` before freeing.
- **`@greptile-apps[bot] @ external_sort.c:1524` (P1)** — single-process `SORT_INDEX_LEAF` path missing scancache cleanup. **Addressed**: `bt_load_heap_scancache_end_for_attrinfo` now called at `external_sort.c:1517` after `sort_listfile_internal` returns.
- **`@greptile-apps[bot] @ external_sort.c:4972` (P1)** — `tmp_sects` uninit + `file_get_num_data_sectors` accumulation. **Resolved as non-bug**: `file_get_num_data_sectors` writes `*n_sectors_out = ...` (assignment), and locals are zero-init at PR head. Greptile flagged based on a misread.
- **`@greptile-apps[bot] @ external_sort.c:4783` (P1)** — `pow(SORT_PX_MERGE_FILES, level + 1)` FP for integer index math. **Author response: "원래 코드가 그래" (existing code style)**. Mirrors pre-existing `sort_merge_run_for_parallel`. Safe in practice — `pow(4, k)` for k ≤ 26 is exactly representable in double.
- **`@hornetmj @ btree_load.c:1081`** — `is_sysop_started` set twice in SERVER_MODE, missing in SA_MODE. **Addressed in commit `42b92007`** "Set is_sysop_started under SERVER_MODE guard after sort".
- **`@shparkcubrid @ btree_load.c:3295`** — `pgbuf_ordered_unfix(old_pgwatcher)` missing on early-exit paths (sector exhausted → `S_END`, non-dealloc error → `S_ERROR`). **Addressed in commit `ab8ca3a`** "Unfix old_pgwatcher on early exits in get_next_vpid". Real page-pin leak fix.
- **`@shparkcubrid @ btree_load.c:3403`** — duplicate check in get_next_vpid. Author kept it ("빼면 오류가 발생해서 넣었습니다") — the duplicate is load-bearing for an error path.
- **`@shparkcubrid @ external_sort.c:1641`** — NIT: `get_fn` assignment placement. **Addressed in commit `e71b21b`** — hoisted into the per-PX setup loop alongside `memcpy`, eliminating per-worker reassignment.
- **`@shparkcubrid @ external_sort.c:5057`** — NIT: `sort_put_result_for_parallel` modification was dead code (only used by SORT_ORDER_BY path). **Addressed in commit `e71b21b`** — reverted to unconditional push/pop.
- **`@shparkcubrid @ btree_load.c:3241`** — NIT: `found` flag unused in `while (!found)` loop where all paths return. **Addressed in commit `44d92db`** — removed flag, `while (true)`.
- **`@youngjinj @ btree_load.h:272`** — forward decl placement. **Addressed in commit `8fbcede`**.
- **`@hornetmj @ btree_load.h:364, btree_load.c:821`** — code-review nits on `bt_load_clear_pred_and_unpack` signature and structural cleanup. **Addressed in commit `8fbcede`**.

## Reconciliation Plan

Executable post-merge (or via `apply reconciliation for PR #7011`). Per affected page:

### [[components/btree]] — index-build subsystem

- **Current state:** documents `xbtree_load_index` and serial sort.
- **Add a new "Parallel index build" subsection** documenting:
  - `SORT_INDEX_LEAF` activation flow: `xbtree_load_index → btree_index_sort → sort_listfile → SORT_EXECUTE_PARALLEL` (when `sort_check_parallelism > 1`).
  - The SORT_ARGS struct now lives in `btree_load.h` (was file-static); enumerate the new fields (`filter_index_info`, `ftab_sets`, `curr_sec`, `curr_pgoffset`).
  - The four newly-extern'd helpers (`bt_load_heap_scancache_start_for_attrinfo`/`end`, `btree_load_filter_pred_function_info`, `bt_load_clear_pred_and_unpack`).
  - `btree_sort_get_next_parallel` + `get_next_vpid` per-thread sector iterator with the `pgbuf_ordered_fix` + `pgbuf_replace_watcher` page-fix protocol; the `ER_PB_BAD_PAGEID` retry on concurrent dealloc; the early-exit unfix invariant from commit `ab8ca3a`.
  - Per-worker XASL filter/func-index re-deserialization (`btree_load_filter_pred_function_info`); rationale = `cache_pred` is mutated during evaluation, sharing across workers would race.
  - Sysop lifecycle: SA_MODE owns sysop in `xbtree_load_index`; SERVER_MODE delegates to `external_sort.c` (single-process at line 1500, parallel at line 4848 inside `sort_merge_run_for_parallel_index_leaf_build`); caller sets `is_sysop_started = true` under SERVER_MODE guard so it can abort/commit later.
- **Callout:** `[!update]` with merge-sha; cite this PR.

### [[components/external-sort]] — sort_listfile orchestration

- **Add an "Index-leaf parallel build" section** at `## Sort modes` documenting:
  - The `SORT_INDEX_LEAF` `SORT_PARALLEL_TYPE` value (existed in enum since PR #5694, fully wired by PR #7011).
  - `sort_merge_run_for_parallel_index_leaf_build` — log₄ tree-merge over per-thread temp files; empty-worker skip; final `btree_create_file` + `sort_put_result_from_tmpfile` on the main thread after merge.
  - `sort_listfile` SERVER_MODE single-process arm wraps scancache + `btree_create_file` + `log_sysop_start` around `sort_listfile_internal`.
  - SA_MODE branch unchanged — owns scancache and sysop in `btree_load.c`.
  - Asymmetry: SA_MODE single-process and SERVER_MODE single-process use different lifecycle owners; document this contract.
- **Callout:** `[!update]` with merge-sha. Add an `[!info]` for the asymmetry.

### [[components/parallel-sort]] — px_sort orchestration

- **Resolve the previously-flagged gap** in [[hot]] cache: `sort_copy_sort_param` is implemented at `external_sort.c:4344-4471` (not `px_sort.c` — the implementation lives next to the consumer).
- **Add note:** `sort_start_parallelism` now has a SORT_INDEX_LEAF arm at `external_sort.c:5248-5310` that allocates per-worker `SORT_ARGS` via `malloc + memcpy`, splits `ftab_sets` across workers, and overrides `get_fn = &btree_sort_get_next_parallel`.
- **Add note:** `sort_end_parallelism` dispatches `sort_merge_run_for_parallel_index_leaf_build` for SORT_INDEX_LEAF.

### [[components/parallel-heap-scan-input-handler]] — ftab_set

- **Update path frontmatter:** `parallel_heap_scan::ftab_set` is now an alias of `parallel_query::ftab_set` (file moved to `src/query/parallel/px_ftab_set.hpp`).
- **Add note:** the class gained move/copy semantics, `append`, `move_from`, and `size()`. Used by both heap-scan input-handler and (post-#7011) the parallel index-build setup in `external_sort.c::sort_start_parallelism`.

### [[components/parallel-query]] — `px_*` umbrella

- **Add `px_ftab_set.hpp` to the file inventory** (currently lives in `parallel_query` namespace alongside `px_interrupt.hpp`, `px_parallel.hpp`, `px_thread_safe_queue.hpp`).
- Note that `parallel_query::ftab_set` is consumed by both `parallel_heap_scan` (via `using` alias) and `external_sort.c` (directly).

### [[components/file-manager]] — file_get_num_data_sectors

- **Add to public API list:** `file_get_num_data_sectors(thread_p, vfid, *n_sectors_out)` — assignment-style (not accumulation). Reads `n_sector_full + n_sector_partial` from the file header.
- **Used by:** `sort_check_parallelism` to decide if parallel build is worthwhile.

### [[components/storage]] — surface for `parallel_query` namespace

- Add a one-liner cross-reference noting that `external_sort.c` and `btree_load.c` now consume `parallel_query::ftab_set` (post-#7011).

### Optional ADR

- **Candidate ADR**: "Parallel index build via SORT_INDEX_LEAF" — covers the architectural choice to (a) reuse the parallel-sort infrastructure rather than build a dedicated parallel-build framework, (b) per-worker XASL re-deserialization rationale, (c) the sysop-ownership-by-mode pattern, (d) the conservative `n_classes == 1` parallelism gate. Judgment call after merge.

## Pages Reconciled

n/a — PR is OPEN. Promote Plan content here on `apply reconciliation for PR #7011`.

## Incidental wiki enhancements

1. **[[hot]]** — annotated the existing follow-up note `sort_copy_sort_param declared in px_sort.h but implementation missing in px_sort.c` to reflect that PR #7011 provides the implementation in `external_sort.c:4344-4471` (the function lives next to its consumer, not in `px_sort.c`). This is a baseline-truth correction — the function is currently *not* in baseline because PR #7011 is open, but the prior wiki claim that the declaration is dangling forever was misleading.

## Deep analysis — supplementary findings

Synthesized from 2 subagent reports. Most greptile P0/P1 findings have been addressed by the author. The findings below are observations beyond bot review.

### Correctness — observations beyond review

1. **SA_MODE / SERVER_MODE single-process asymmetry.** `sort_listfile`'s SA_MODE branch (`external_sort.c:1559-1561`) skips the scancache + sysop + `btree_create_file` setup that the SERVER_MODE single-process branch now performs. This is **not a bug** — `btree_load.c::xbtree_load_index` keeps its own setup under `#if !defined(SERVER_MODE)` (line 1078-1085). But ownership of scancache lifecycle differs by mode: a future contributor adding an SA_MODE consumer of `sort_listfile(..., SORT_INDEX_LEAF)` outside `xbtree_load_index` would silently miss the setup. Worth a callout in the wiki.
2. **`sort_args->cur_class` ↔ `ftab_sets[cur_class]` parallel-array invariant** is not asserted. `sort_start_parallelism` always pushes one entry (possibly empty) per `n_classes` to keep the index aligned, but no `assert(sort_args->ftab_sets->size() == sort_args->n_classes)` exists at the consumer side. A future change adding a class without extending `ftab_sets` would index UB.
3. **`get_next_vpid` ftab-page skip** at `btree_load.c:3269-3273` compares `vpid_out->pageid == hfid->vfid.fileid && vpid_out->volid == hfid->vfid.volid`. Conflates a `pageid` with a `fileid` — they're both `int32_t` and a heap file's first page id equals its `fileid` by convention. Author relies on the convention but never asserts it.
4. **`prev_oid` in `btree_sort_get_next_parallel`** (declared and assigned but never read) appears to be dead code copy-pasted from the serial `btree_sort_get_next`. Drop.

### Code concerns / smells

5. **`btree_sort_get_next_parallel` declared in `external_sort.h`** (`:165`) rather than `btree_load.h`. The function is *defined* in `btree_load.c`; its only caller takes its address in `external_sort.c::sort_start_parallelism`. Cleaner placement is in `btree_load.h` (since `external_sort.c` already includes it).
6. **`sort_merge_run_for_parallel_index_leaf_build` lacks `static`** at definition (`external_sort.c:4733`) despite being only used internally and having a static forward decl at line 303-304. Add `static` to match.
7. **`sort_put_result_for_parallel` defensive guards** at `external_sort.c:5063, 5090` are dead code for SORT_INDEX_LEAF (function isn't dispatched for that path). Author's reply confirms — they're kept defensively. Worth a `// Defensive: SORT_INDEX_LEAF currently dispatches sort_put_result_from_tmpfile directly` comment.
8. **Stale comment** at `external_sort.c:5314` — `/* not implemented yet (group by, analytic fuction) */` (typo "fuction"). PR removed "create index" from the parenthetical correctly but didn't fix the typo.
9. **`btree_index_sort` still passes hard-coded `0`** for parallelism hint at `btree_load.c:3228`: `sort_listfile (..., 0 /* TODO - support parallelism */, ...)`. The TODO predates this PR but the comment is now lying — parallelism is decided inside `sort_check_parallelism` based on `sort_param->px_type == SORT_INDEX_LEAF` heuristics. Update comment.
10. **`btree_index_sort` still passes `&btree_sort_get_next`** (the serial getter) at the same site. The parallel getter is swapped in by `sort_start_parallelism`. Functionally fine but readers will trace the wrong code path — comment "may be overridden in sort_start_parallelism for SORT_INDEX_LEAF" would help.
11. **`pow(SORT_PX_MERGE_FILES, level + 1)` floating-point for integer math** at `external_sort.c:4766, 4769, 4783`. Mirrors the pre-existing `sort_merge_run_for_parallel`. Author defended. Safe in practice (`pow(4, k)` for k ≤ 26 is exactly representable). Stylistically replace with `1 << (2*level)` or a static `inline int ipow4(int)`. Future cleanup.
12. **Manual `placement_new` + `~vector()` + `free_and_init` for `ftab_sets`** is correct but fragile. An exception inside `vector::push_back` (e.g. `bad_alloc`) would leak the `malloc`'d block before placement_new completes. Wrapping in a smart-pointer-equivalent or using `db_private_alloc` + RAII would be cleaner. Not a current bug.
13. **`std::vector` in a public header** — `SORT_ARGS::ftab_sets` is `std::vector<parallel_query::ftab_set> *` (`btree_load.h`). The header is included by both C and C++ TUs (e.g. `btree.c`). The author wraps the relevant lines in `/* *INDENT-OFF* */` comments suggesting awareness, but if any C TU pulls `btree_load.h`, the `std::vector` line will fail to compile. Verify all includers are C++. Quick check needed. If a C TU includes it, the fix is to make the parallel-state fields opaque (`void *ftab_sets`).
14. **Parent's `func_index_info` unpack is wasted in parallel SERVER_MODE.** `xbtree_load_index` unpacks `func_index_info` once on the parent (`btree_load.c:977-990`); for parallel SERVER_MODE the parent never iterates the heap, so the unpack is unused. Keeps the SA_MODE call path clean but minor inefficiency.

### Performance

15. **Per-worker XASL re-unpack cost.** O(parallel_num) × tree size. Typical filter `WHERE col > N AND col < M` produces a few-KB unpack arena per worker. At default `parallel_num` (single-digit), no concern. Cost grows linearly with `prm_get_max_parallelism`.
16. **`pow()` calls in tight per-merge-slot code.** Each merge slot does up to 4 `pow` calls. Negligible at typical `parallel_num` but the kind of thing that compiles to a function call into libm.

### Resolves baseline gap

17. **`sort_copy_sort_param` implementation provided.** [[hot]] cache previously noted "declared in `px_sort.h` but implementation missing in `px_sort.c`". The function lives at `external_sort.c:4344-4471` (next to consumer, not in `px_sort.c` — sensible placement).

## Baseline impact

- Before: `175442fc858bd0075165729756745be6f8928036`
- After: `175442fc858bd0075165729756745be6f8928036` (unchanged — PR is open)
- Bump triggered: `false`
- Logged: see [[log]] entry `[2026-04-26] pr-ingest PR #7011`.

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/7011
- Jira: [CBRD-26678](https://jira.cubrid.org/browse/CBRD-26678)
- Components touched: [[components/btree]], [[components/external-sort]], [[components/parallel-sort]], [[components/parallel-heap-scan-input-handler]], [[components/parallel-heap-scan]], [[components/parallel-query]], [[components/file-manager]], [[components/storage]]
- Sources: [[sources/cubrid-src-storage]], [[sources/cubrid-src-query-parallel]]
- Adjacent PRs:
  - [[prs/PR-7062-parallel-scan-all-types]] — generalizes parallel scan over heap/list/index types; shares the `parallel_heap_scan → parallel_scan` namespace migration pattern that this PR's `parallel_query::ftab_set` move complements.
  - [[prs/PR-6911-parallel-heap-scan-io-bottleneck]] — the upstream change that introduced `ftab_set::split` and the sector-pre-allocation pattern this PR reuses for index-build heap partitioning.
  - [[prs/PR-6753-optimizer-histogram-support]] — also OPEN, also touches `query/parallel` plumbing — minor merge-conflict surface possible.
