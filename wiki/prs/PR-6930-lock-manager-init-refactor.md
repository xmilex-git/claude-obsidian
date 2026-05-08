---
type: pr
pr_number: 6930
pr_url: "https://github.com/CUBRID/cubrid/pull/6930"
repo: "CUBRID/cubrid"
state: MERGED
is_draft: false
author: "hgryoo"
created_at: 2026-03-23
merged_at: 2026-05-07
closed_at:
merge_commit: "5e12a293c609a5d99c39b4c81a00b89b9ef91662"
base_ref: "develop"
head_ref: "CBRD-26478_refactor"
base_sha: "0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0"
head_sha: "5e12a293c609a5d99c39b4c81a00b89b9ef91662"
jira: "CBRD-26478"
files_changed:
  - "src/transaction/lock_manager.c"
related_components:
  - "[[components/lock-manager]]"
  - "[[components/deadlock-detection]]"
related_sources:
  - "[[sources/cubrid-src-transaction]]"
ingest_case: "c"
triggered_baseline_bump: true
baseline_before: "0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0"
baseline_after: "5e12a293c609a5d99c39b4c81a00b89b9ef91662"
reconciliation_applied: true
reconciliation_applied_at: 2026-05-08
incidental_enhancements_count: 1
tags:
  - pr
  - cubrid
  - lock-manager
  - refactor
  - initialization
created: 2026-05-08
updated: 2026-05-08
status: merged
---

# PR-6930-lock-manager-init-refactor

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `MERGED` · **Author:** `@hgryoo` · **Merge commit:** `5e12a293c`
> **Base → Head:** `develop` (`0be6cdf6e`) → `CBRD-26478_refactor` (`5e12a293c`)

> [!note] Ingest classification: case (c)
> Merge commit `5e12a293c` is a direct descendant of baseline `0be6cdf6e` on `develop` (single squash-merge). PR-reconciliation applied during this ingest; baseline bumped to the merge commit.

## Summary

Refactor of `lock_manager.c` initialization/finalization. Centralizes every tunable into an internal `LK_CONFIG` value, tracks per-substructure init in an `LK_INIT_STATE`, and replaces statically-sized deadlock buffers (`TWFG_edge_block[]`, `victims[]`) with lock-manager-owned heap allocations sized from config. Behavior under default settings is unchanged; the goal is to make init failures cleanly recoverable and to give a single seam for future per-deployment tuning of TWFG / victim-buffer sizing.

## Motivation

Pre-PR `lock_initialize` was a flat function that mixed parameter reading (`envvar_get`), table-static C arrays (`TWFG_edge_block[LK_MID_TWFG_EDGE_COUNT]`, `victims[LK_MAX_VICTIM_COUNT]`), preprocessor `#define`s (`LK_MIN_TWFG_EDGE_COUNT`, `LK_MAX_VICTIM_COUNT`, …), and field-level initialization on `lk_Gl`. `lock_finalize` had to mirror this, but had two latent bugs the refactor closes:

1. **Partial-init cleanup.** If `lock_initialize_tran_lock_table` failed mid-loop (after some entries had `pthread_mutex_init`'d but before all of them), `lock_finalize` would iterate `0..lk_Gl.num_trans` — but `num_trans` was incremented per successful init, so the cleanup loop matched the actual work done only by lucky ordering. The new code records `tran_lock_table_initialized_count` explicitly so finalize destroys exactly the mutexes that were created.
2. **Unconditional `pthread_mutex_destroy` on `DL_detection_mutex`.** Pre-refactor `lock_finalize` always called `pthread_mutex_destroy(&lk_Gl.DL_detection_mutex)` even on the failure path where `lock_initialize_deadlock_detection` had not yet run `pthread_mutex_init`. The new `init_state.deadlock_detection_initialized` flag gates the destroy.

Both bugs were latent under the default `lock_initialize()` path because allocations rarely fail at boot, but they were real and reachable via constructed OOM scenarios.

## Changes

### Structural

- **New types** (file-local, in `src/transaction/lock_manager.c`):
  - `LK_CONFIG` — 16 fields covering transaction-table sizing, object-lock hash sizing, deadlock-detector resources, and diagnostics flags (`verbose_mode`, `dump_level`).
  - `LK_INIT_STATE` — four flags/counters tracking which substructures are initialized: `tran_lock_table_initialized`, `tran_lock_table_initialized_count`, `object_lock_structures_initialized`, `deadlock_detection_initialized`.
- **`LK_GLOBAL_DATA` (`lk_Gl`)** gains `LK_CONFIG config` and `LK_INIT_STATE init_state`. Removes `max_obj_locks`, `num_trans`, `verbose_mode`, `dump_level` fields (now in `config`).
- **New file-static functions:**
  - `lock_make_default_config()` — pure: returns hard-coded defaults.
  - `lock_make_runtime_config()` — derives `num_trans`-dependent fields, enforces `min_twfg_edge_count < mid_twfg_edge_count <= max_twfg_edge_count` invariant (prevents heap overflow in `lock_add_WFG_edge`'s 2-stage TWFG buffer expansion), and applies `LK_VERBOSE_SUSPENDED` / `LK_DUMP_LEVEL` env overrides.
  - `lock_initialize_with_config (const LK_CONFIG *)` — copies config into `lk_Gl.config`, then runs init steps in fixed order.
  - `lock_initialize_object_lock_structures` — replaces previous pair (`lock_initialize_object_hash_table` + `lock_initialize_object_lock_entry_list`); now atomic w.r.t. error handling — if freelist init fails, the hash table is destroyed before returning.
  - `lock_finalize_object_lock_structures`, `lock_finalize_deadlock_detection` — explicit per-substructure finalize; gated on init_state flags.
- **Removed file-static storage:**
  - `static LK_WFG_EDGE TWFG_edge_block[LK_MID_TWFG_EDGE_COUNT]` → `lk_Gl.TWFG_edge_storage` (heap-allocated in `lock_initialize_deadlock_detection`).
  - `static LK_DEADLOCK_VICTIM victims[LK_MAX_VICTIM_COUNT]` → `lk_Gl.victims` (heap-allocated).
- **Removed macros / file-static constants:**
  - `LK_MIN_OBJECT_LOCKS`, `LK_RES_RATIO`, `LK_ENTRY_RATIO`, `LK_MORE_RES_COUNT`, `LK_MORE_ENTRY_COUNT`, `LK_MAX_VICTIM_COUNT`, `LK_MIN_TWFG_EDGE_COUNT`, `LK_MID_TWFG_EDGE_COUNT`, `LK_MAX_TWFG_EDGE_COUNT`, `LOCK_TRAN_LOCAL_POOL_MAX_SIZE` — all now `LK_CONFIG` fields.
- **`lock_initialize` API unchanged.** Public signature `int lock_initialize(void)` preserved. Internally it builds a runtime config and calls `lock_initialize_with_config`.
- **`lock_finalize` ordering documented as reverse-init.** Calls daemon destroy → `lock_finalize_deadlock_detection` → `lock_finalize_object_lock_structures` → `lock_finalize_tran_lock_table`. Final step resets `lk_Gl.config` and `lk_Gl.init_state` to defaults so a subsequent `lock_initialize` starts clean.

### Per-file notes

- `src/transaction/lock_manager.c` — sole file touched. +364 / −224 LOC. Net +140. Breaks `lock_initialize`/`lock_finalize` into the smaller surface listed above; updates every reader of the removed fields/macros to reference `lk_Gl.config.*` or `lk_Gl.victims[…]`. ([[components/lock-manager]] · [[components/deadlock-detection]])

### Behavioral

- **Default-path semantics unchanged.** All defaults (300 victims, MIN/MID 200/1000 TWFG edges, 10000 initial object locks, 0.1 res/entry ratios, 10-entry transaction local pool, etc.) carry over from the removed `#define`s into `lock_make_default_config()` byte-for-byte.
- **Daemon start gated on `config.start_deadlock_detector`.** Pre-refactor the daemon was always started in `lock_initialize`; post-refactor `lock_initialize_with_config` checks the config flag. Default is `true` so server boot still starts the daemon — but tests / non-server callers can now construct a config with `start_deadlock_detector = false` and skip it.
- **Daemon pointer null-out on destroy.** `lock_deadlock_detect_daemon_destroy()` now sets `lock_Deadlock_detect_daemon = NULL` after `cubthread::get_manager()->destroy_daemon(...)`. This makes a second `lock_finalize()` (e.g. error path → final cleanup) safe; previously the second destroy would have dereferenced a freed daemon handle.
- **TWFG buffer is now heap-owned.** `lock_detect_local_deadlock` formerly assigned `lk_Gl.TWFG_edge = &TWFG_edge_block[0]` from a `.bss` array; now it assigns `lk_Gl.TWFG_edge = lk_Gl.TWFG_edge_storage` from a heap allocation made at init. The "did we grow beyond mid?" check in cleanup changes from `lk_Gl.max_TWFG_edge > LK_MID_TWFG_EDGE_COUNT` to `lk_Gl.TWFG_edge != lk_Gl.TWFG_edge_storage` — pointer-identity rather than counter.
- **`victims` are heap-owned and zero-initialized.** `lock_initialize_deadlock_detection` `memset`s the new buffer; pre-refactor the static `.bss` array was zero-initialized by C startup. Same observable state, different allocation site.
- **TWFG count invariants enforced.** `lock_make_runtime_config` clamps `mid_twfg_edge_count = min + 1` if `mid <= min`, and `max = mid + 1` if `max < mid`. Under defaults (200 / 1000 / `MAX_NTRANS²`), the invariant holds with margin; the clamp matters only if a future caller hands `lock_initialize_with_config` a custom config — without it, `lock_add_WFG_edge`'s 2-stage expansion (200 → 1000 → max) would heap-overflow when `mid <= min`.
- **Failure rollback in `lock_initialize_object_lock_structures`.** If `lf_freelist_init` fails, the function now calls `lk_Gl.m_obj_hash_table.destroy()` before returning. Pre-refactor it left the hash table allocated — a subtle leak on the OOM-during-boot path.
- **Failure rollback in `lock_initialize_deadlock_detection`.** OOM at `TWFG_edge_storage` malloc frees `TWFG_node`; OOM at `victims` malloc frees both `TWFG_edge_storage` and `TWFG_node` before returning. Pre-refactor there was no rollback because the failing allocations didn't exist (the structures were `.bss` static arrays).

### New surface (no existing wiki reference)

- `LK_CONFIG`, `LK_INIT_STATE` — file-local types; not part of public API. No dedicated wiki page warranted; documented inline on [[components/lock-manager]].

## Review discussion highlights

- **`max_twfg_edge_count` heap overflow** (greptile P1, fixed at 96fe7b0). Original `lock_make_runtime_config` unconditionally overwrote `max_twfg_edge_count = num_trans * num_trans`; if a future custom config set `mid_twfg_edge_count > num_trans²`, `lock_add_WFG_edge`'s second-stage `malloc(mid)` followed by index access into the `max`-sized world would walk off the end. The fix added the `if (max < mid) max = mid + 1` clamp.
- **`TWFG_edge` dangling pointer** (greptile P1). Reviewer noted that after `lock_finalize_deadlock_detection` frees `TWFG_edge_storage`, `lk_Gl.TWFG_edge` still pointed at the freed region. Author accepted; the final shipped code sets `lk_Gl.TWFG_edge = NULL` explicitly after the storage free.
- **`pthread_mutex_destroy` on uninit mutex** (greptile P1). Previously addressed: `lock_finalize_deadlock_detection` now gates `pthread_mutex_destroy(&lk_Gl.DL_detection_mutex)` on `init_state.deadlock_detection_initialized`.
- **`lock_tune_init_config` redundancy** (hornetmj). Reviewer pointed out that an earlier draft had a separate "tune" pass that re-applied defaults pointlessly. Author folded the validation/tune logic into `lock_make_runtime_config` (commit `eb32d59`) and noted intent to split it back out if needed.
- **Pass-by-pointer for config** (hornetmj). Earlier draft passed `LK_CONFIG` by value into multiple helpers; reviewer asked for a single pointer copy at `lock_initialize_with_config` entry. Final code copies once: `lk_Gl.config = *config;` in `lock_initialize_with_config`.
- **`LK_DUMP` / `CUBRID_DEBUG` macros kept.** hyahong asked whether the diagnostic gates were still in use; hgryoo answered that `LK_DUMP` is reachable via env var when needed for issue debugging and that `CUBRID_DEBUG` cleanup is deferred to a separate sweep. Both kept verbatim.

## Reconciliation Plan

Applied during this ingest — see Pages Reconciled below.

## Pages Reconciled

- [[components/lock-manager]] — added section "Initialization (since PR #6930)" describing `LK_CONFIG` / `LK_INIT_STATE` layering and the `lock_make_default_config → lock_make_runtime_config → lock_initialize_with_config` chain. Public API unchanged. `[!update]` callout cites PR #6930.
- [[components/deadlock-detection]] — replaced "static `TWFG_edge_block[]` / `victims[]`" claims with the new heap-owned `lk_Gl.TWFG_edge_storage` / `lk_Gl.victims` layout. Added invariant note for `min < mid <= max` TWFG-edge counts. `[!update]` callout cites PR #6930.

## Incidental wiki enhancements

- [[components/lock-manager]] — `[!gap]` filled: added explicit note that `lock_finalize` calls `lock_deadlock_detect_daemon_destroy` first and that this null-checks `lock_Deadlock_detect_daemon` post-PR (which makes idempotent finalize safe). The pre-PR wiki did not document finalize order at all.

## Baseline impact

- Before: `0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0`
- After: `5e12a293c609a5d99c39b4c81a00b89b9ef91662`
- Bump triggered: `true`
- Logged: [[log]] under `[2026-05-08] baseline-bump | 0be6cdf6 → 5e12a293`

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/6930
- Jira: CBRD-26478
- Components: [[components/lock-manager]] · [[components/deadlock-detection]]
- Sources: [[sources/cubrid-src-transaction]]
