---
type: source
source_kind: session-note
title: "QMGR_TEMP_FILE *tfile — Role Analysis"
origin_path: "/home/cubrid/dev/cubrid/.claude/sessions/2026-04-28-tfile-role-analysis.md"
origin_branch: "parallel_scan_all"
origin_date: 2026-04-28
related_pr:
  - "[[prs/PR-6981-parallel-hash-join-sector-split|PR #6981]]"
  - "[[prs/PR-7062-parallel-scan-all-types|PR #7062]]"
related_components:
  - "[[components/query-manager]]"
  - "[[components/list-file]]"
  - "[[components/parallel-hash-join-task-manager]]"
tags:
  - source
  - cubrid
  - parallel-query
  - tfile
  - design-analysis
created: 2026-04-28
updated: 2026-04-28
status: active
---

# `QMGR_TEMP_FILE *tfile` — Role Analysis (2026-04-28 session note)

> [!info] Origin
> External session note: `/home/cubrid/dev/cubrid/.claude/sessions/2026-04-28-tfile-role-analysis.md`
> Captured on branch `parallel_scan_all` (PR #7062 head) the same day baseline was bumped to `0be6cdf6` (PR #6981 merge).
> Decision-prep: whether to integrate PR #6981's `qfile_collect_list_sector_info` + `tfiles[]` parallel-array pattern into the open PR #7062's `px_scan_input_handler_list`.

## One-line summary

`tfile` is essentially required only for **(1) membuf entry lookup** and **(2) page-type discrimination on free** (membuf vs buffer pool). For **disk pages alone**, a `vpid` is sufficient for both read and free.

## `qmgr_temp_file` structure (baseline truth)

```c
struct qmgr_temp_file {
  QMGR_TEMP_FILE *next, *prev;
  FILE_TYPE temp_file_type;
  VFID temp_vfid;          /* disk file identifier */
  int membuf_last;         /* -1 = membuf unused */
  PAGE_PTR *membuf;        /* in-memory page array; NULL when absent */
  int membuf_npages;
  QMGR_TEMP_FILE_MEMBUF_TYPE membuf_type;
  bool preserved;
  bool tde_encrypted;
};
```

## Four code paths that consume `tfile`

### 1. `qmgr_get_old_page*()` — Read

```c
if (vpid->volid == NULL_VOLID)        /* membuf-encoded VPID */
    page = tfile->membuf[vpid->pageid];   /* tfile required */
else                                   /* disk page */
    page = pgbuf_fix(vpid, ...);          /* tfile not consulted */
```

- Disk page: **`tfile` not required.** A `vpid` alone is enough for `pgbuf_fix`.
- Membuf page: **`tfile` required.** The `(NULL_VOLID, pageid)` encoding makes `pageid` the membuf index — so the worker must hold a pointer to the owning tfile.

### 2. `qmgr_free_old_page*()` — Free

```c
if (tfile == NULL) {
    pgbuf_unfix(page);                  /* always buffer-pool unfix */
    return;
}
page_type = qmgr_get_page_type(page, tfile);
if (page_type == QMGR_TEMP_FILE_PAGE)
    pgbuf_unfix(page);                  /* disk page */
/* else QMGR_MEMBUF_PAGE → no-op (membuf is not in the buffer pool) */
```

- `qmgr_get_page_type` decides via pointer-range check (`page_p` ∈ `tfile->membuf[0..membuf_last]`).
- A **dependent list** tfile always has `membuf == NULL` / `membuf_last < 0`, so the page-type check is effectively trivial there → behaviour collapses to `pgbuf_unfix`.
- For disk pages alone, calling `qmgr_free_old_page` with `tfile = NULL` produces the same observable effect.

### 3. `qmgr_set_dirty_page` / `qmgr_get_new_page` — Write

Not on the scan path (read-only consumers). `px_scan_input_handler_list` / `px_scan_slot_iterator_list` do not call these.

### 4. `qfile_get_tuple()` and other list helpers

- Walking the overflow chain pulls/releases pages internally using `tfile_vfid`.
- `px_scan_slot_iterator_list.cpp:130` (PR #7062 branch) calls `qfile_get_tuple(..., m_list_id)`.
- When `m_list_id` is the base and the page came from a *dependent* list, the internal traversal will use the *base's* tfile — semantic correctness depends on whether that traversal needs the dependent's tfile context.

## `qfile_connect_list` invariant

`list_file.c:3130` (`qfile_connect_list (base_list_id, append_list_id)`):

```c
assert (append_list_id->tfile_vfid->membuf == NULL);
```

> [!key-insight] Membuf is exclusive to the head of a dependent-list chain
> A dependent list's `tfile_vfid` never carries a membuf array. Only the *first* `QFILE_LIST_ID` in a chain can (and only if the list spilled to memory in the first place). Consequence:
> - The membuf-handling worker (e.g. PR #6981's CAS-claim winner; or "worker 0" in any other sector-distribution scheme) only needs the *base* tfile.
> - A dependent-list page never requires the membuf-aware path of `qmgr_get_page_type` — the per-thread `m_current_tfile` pointer recorded by `get_next_page` is being used purely for **API contract** (pass the owning tfile back to `qmgr_free_old_page`), not because the membuf check actually fires for it.

## `px_scan_input_handler_list` (PR #7062 branch) — current evaluation

> [!warning] Off-baseline content
> The `px_scan_input_handler_list*` paths discussed below exist only on PR #7062's `parallel_scan_all` branch. They are not part of baseline `0be6cdf6` and do not appear in any component page in this wiki. Numbers below are the analysis author's reading of the PR-branch code.

`init_on_main()` collects sectors from base + all dependents into a single `ftab_set`, then partitions across workers — but `fetch`/`free` always use the base's `m_tfile_vfid` (`px_scan_input_handler_list.cpp:202, 208` on the branch).

| Path | Base page | Dependent page (using base tfile) | Correctness |
|------|-----------|-----------------------------------|-------------|
| Read (disk) | OK | OK (only `vpid` matters) | **OK** |
| Read (membuf) | OK (`volid = NULL_VOLID`) | n/a (no membuf in dependents) | OK |
| Free (disk) | OK | OK (page is outside base membuf range → `TEMP_FILE_PAGE` either way) | **OK** |
| `qfile_get_tuple` overflow chain | OK | **Potential risk** — list_id passed is base; internal traversal runs in base tfile/list_id context | Needs verification |

**Tentative conclusion.** Plain page fetch/free with a single base-list tfile is unlikely to regress against dependents. PR #6981's introduction of `tfiles[]` + `m_current_tfile` was driven mostly by **overflow-continuation chain traversal** (multi-page tuples) and **semantic consistency under interleaved base/dependent lists**, rather than any failure of the basic page API.

## Decision implications for PR #7062 integration

### Option B — adopt PR #6981's full pattern (`qfile_collect_list_sector_info` + `tfiles[]` parallel array)

- **Solid wins:**
  - Removes ~50+ duplicated lines (one helper call replaces ad-hoc collection).
  - Same ownership model as parallel hash join — easier later refactor / unification.
  - `tfiles[]` overhead is one pointer per sector — effectively zero.
- **Conditional value:**
  - May make overflow-chain handling for *dependent-list* pages strictly more correct (requires Option B's `m_current_tfile` to thread through `qfile_get_tuple`).
- **Risk:** `ftab_set::split` and the `tfiles[]` array must be partitioned in lock-step. Mechanical change, low risk.

### Option B' — simplified, no per-sector tfile tracking

A reduced variant motivated by the observation that disk-page read/free does not need the dependent tfile:

- Workers don't track per-sector tfile; either pass `NULL` to `qmgr_free_old_page` or always pass base tfile (read/free still correct for disk pages).
- BUT `px_scan_slot_iterator_list`'s overflow path still calls `qfile_get_tuple(..., list_id)`. To preserve semantic accuracy without `tfiles[]`, a per-sector → list_id map is still needed (i.e. the same shape as `tfiles[]`, just at the list_id rather than the tfile level).

→ **Recommended approach.** Adopt Option B (full pattern), but verify by 1:1 enumeration of `m_current_tfile` consumers in PR #6981's `split_task::execute`. If the only effective use is page fetch/free on disk pages, drop to Option B'. The deciding signal is whether overflow-continuation traversal observably needs the dependent's tfile.

## Suggested follow-up actions

1. Enumerate every `m_current_tfile` consumption point in PR #6981's `split_task::execute()` (page fetch / page free / overflow chain) — see [[components/parallel-hash-join-task-manager#Page Cursor — Lock-free sector bitmap walk (`get_next_page`)]] for the recorded use sites.
2. Inspect `px_scan_slot_iterator_list::next_slot` overflow path (PR #7062 branch) to see how it currently handles dependent-list pages.
3. Decide Option B vs B' based on (1)+(2).

## Code references (verified against baseline `0be6cdf6`)

| Path | Subject |
|------|---------|
| `src/query/query_manager.c:201` | `qmgr_get_page_type` (membuf-vs-disk pointer-range check) |
| `src/query/query_manager.c:2520` | `qmgr_get_old_page` |
| `src/query/query_manager.c:2582` | `qmgr_free_old_page` |
| `src/query/query_manager.c:2618` | `qmgr_get_old_page_read_only` |
| `src/query/query_manager.h` | `struct qmgr_temp_file` |
| `src/query/list_file.c:3130` | `qfile_connect_list` — membuf-NULL assert on dependent list |
| `src/query/list_file.c:7085` | `qfile_collect_list_sector_info` (added by PR #6981) |
| `src/query/parallel/px_hash_join/px_hash_join_task_manager.cpp` | `m_current_tfile` use sites (PR #6981) |
| `src/query/parallel/px_scan/px_scan_input_handler_list.cpp:73-120` *(branch only)* | Integration target on `parallel_scan_all` |
| `src/query/parallel/px_scan/px_scan_slot_iterator_list.cpp:130` *(branch only)* | Overflow-chain entry point on `parallel_scan_all` |

## Cross-references

- [[prs/PR-6981-parallel-hash-join-sector-split]] — the source pattern (merged, baseline `0be6cdf6`).
- [[prs/PR-7062-parallel-scan-all-types]] — the OPEN PR being decided on. **Note:** because PR #7062 is open, no component-page edit is induced by this analysis; baseline-truth claims it surfaced are applied incidentally to [[components/query-manager]] and [[components/list-file]].
- [[components/parallel-hash-join-task-manager]] — `m_current_tfile` recording.
- [[components/list-file]] — `QFILE_LIST_SECTOR_INFO` and `qfile_collect_list_sector_info`.
