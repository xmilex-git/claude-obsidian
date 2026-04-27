---
created: 2026-04-23
type: meta
title: "Operation Log"
updated: 2026-04-27
tags:
  - meta
  - log
status: evergreen
related:
  - "[[index]]"
  - "[[hot]]"
  - "[[overview]]"
  - "[[sources/_index]]"
---

# Operation Log

Navigation: [[index]] | [[hot]] | [[overview]]

Append-only. New entries go at the TOP. Never edit past entries.

Entry format: `## [YYYY-MM-DD] operation | Title`

Parse recent entries: `grep "^## \[" wiki/log.md | head -10`

---

## [2026-04-27] pr-reconcile + baseline-bump | 65d69154 → cc563c7 (PR #7011)

[[prs/PR-7011-parallel-index-build|PR #7011]] **merged** at 05:20Z as commit `cc563c7fd90521393781d8440bf5144d2566ff71` — direct child of prior baseline `65d69154` on `develop` (case c). PR #7011 was originally ingested OPEN on 2026-04-26 against baseline `175442fc`; reconciliation plan was deferred until merge. This entry promotes the plan and bumps the baseline.

Pages reconciled (all 7 received `[!update]` callouts citing PR #7011 + merge sha `cc563c7f`):
- [[components/btree]] — added "Parallel index build (`SORT_INDEX_LEAF`)" subsection under Bulk Loading: dispatch chain, public `SORT_ARGS` fields, newly-extern'd helpers (`bt_load_heap_scancache_*`, `bt_load_clear_pred_and_unpack`, `btree_load_filter_pred_function_info`, `btree_sort_get_next_parallel`), `FILTER_INDEX_INFO`, `get_next_vpid` page-fix protocol incl. `ab8ca3a` early-exit unfix, per-worker XASL re-deserialization rationale (`cache_pred` mutation), sysop ownership by mode, conservative `n_classes == 1` gate.
- [[components/external-sort]] — added "Index-leaf parallel build (`SORT_INDEX_LEAF`)" section: `sort_merge_run_for_parallel_index_leaf_build` log₄ tree-merge, empty-worker skip, scancache wrap on single-process arm, SA/SERVER asymmetry `[!info]`, two silent fixes (`px_sort_param == NULL` always-true, `malloc → calloc` for `file_contents.num_pages`). Updated SORT_PARALLEL_TYPE table row.
- [[components/parallel-sort]] — annotated `sort_copy_sort_param` row with implementation location (`external_sort.c:4344-4471`, not `px_sort.c`). Added "`SORT_INDEX_LEAF` dispatch (PR #7011)" section.
- [[components/parallel-heap-scan-input-handler]] — frontmatter `path:` updated to `src/query/parallel/px_ftab_set.hpp`. Namespace migration callout. Method table extended with PR #7011 additions (dtor, copy/move ctors+assignment, `append`, `move_from`, `size`).
- [[components/parallel-query]] — `px_ftab_set.hpp` added to `key_files`; `parallel_query::ftab_set` row added to Sub-Components table.
- [[components/file-manager]] — `file_get_num_data_sectors` added to `public_api` frontmatter; "Sector-count helper" subsection added.
- [[components/storage]] — `[!update]` callout under "Parallel Sort Interface" noting new consumers of `parallel_query::ftab_set`.

PR page updates: [[prs/PR-7011-parallel-index-build]] frontmatter set to `state: MERGED`, `ingest_case: c`, `triggered_baseline_bump: true`, `baseline_before: 65d69154`, `baseline_after: cc563c7`, `reconciliation_applied: true`, `status: merged`, `updated: 2026-04-27`. Reconciliation Plan retained verbatim for audit; "Pages Reconciled" section now contains the applied summary.

Verification: between original snapshot head `44d92db64` and final head `6f5ca7ae2` the only landed commit is `6f5ca7a` (`Merge branch 'CUBRID:develop' into parallel_index_build`). `git diff` on the 9 PR-touched files shows no logic changes — original code analysis (2 parallel deep-read subagents on 2026-04-26) remains accurate. Drift in `cubrid/CMakeLists.txt` and `btree_load.c::online_index_builder` is from the develop-merge pulling in PR #7095 (`thread_manager_impl`), unrelated to PR #7011's contribution.

Baseline hash updated in `CLAUDE.md` and `wiki/hot.md`.

## [2026-04-27] baseline-bump | 175442fc → 65d69154 (PR #7049)

Triggered by [[prs/PR-7049-parallel-buildvalue-heap|PR #7049]] ingest (case c — merge commit `65d6915` is direct child of prior baseline `175442fc` on `develop`). Baseline hash updated in `CLAUDE.md` and `wiki/hot.md`.

Pages reconciled:
- [[components/parallel-heap-scan]] — RESULT_TYPE table + selection block: `COUNT_DISTINCT` → `BUILDVALUE_OPT`; broadened scope description; `> [!update]` callout.
- [[components/parallel-heap-scan-result-handler]] — major rewrite: RESULT_TYPE taxonomy table, full specialization section heading + table, "Execution Path" subsection with new write/finalize/read flow; new "Cross-thread DB_VALUE ownership" subsection documenting heap-0 ↔ private-heap dance; `is_buildvalue_opt_supported_function` whitelist documented; MIN/MAX-DISTINCT shortcut documented.
- [[components/parallel-heap-scan-task]] — `MERGEABLE_LIST / COUNT_DISTINCT` → `MERGEABLE_LIST / BUILDVALUE_OPT` rename throughout; `> [!update]` noting new `er_errid()` propagation after `write_initialize`.
- [[components/parallel-heap-scan-support]] — checker section: documented `possible_flags` enum (with `CANNOT_BUILDVALUE_OPT` rename), `is_buildvalue_opt_supported_function` 12-aggregate whitelist, `ACCESS_SPEC_FLAG_BUILDVALUE_OPT` setting on success.
- [[components/aggregate-analytic]] — added "Parallel Heap Scan Fast Path (BUILDVALUE_OPT)" subsection cross-linking to result-handler.

## [2026-04-27] pr-ingest-deep | PR #7049 — Support avg, sum on parallel heap scan (MERGED, 9 files, case c)
- Ingested CUBRID upstream [PR #7049](https://github.com/CUBRID/cubrid/pull/7049) "[CBRD-26711] Support avg,sum function on parallel heap scan" by `@xmilex-git` (the user themselves). State `MERGED` at 2026-04-27T04:52:40Z. Base `develop@2be90e6dd`, head `parallel_buildvalue_heap@c047445c8`. Merge commit `65d6915437eb6217ab0050939c6ad63f0d509735`. 9 files, +424/−158 (582 LOC). Largest: `px_heap_scan_result_handler.cpp` (443 changed). 13 commits squash-merged. 17 inline review comments (mostly greptile P1/P2 — addressed via 6 "greptile review apply" commits + final review-feedback commit).
- **Classification: case (c)** — merge commit is direct child of prior baseline `175442fc` on `develop`. Full PR-reconciliation applied immediately + baseline bumped.
- **Scale rule check**: 9 files, 582 LOC, largest single-file change 443 LOC — under all thresholds (20 / 3000 / 500). Read in main thread without dispatching parallel subagents.
- **PR page**: [[prs/PR-7049-parallel-buildvalue-heap]] with full structural + behavioral analysis. Pages Reconciled section enumerates 7 component-page edits.
- **Goal**: extend parallel heap scan's BUILDVALUE_PROC fast path from COUNT-only to the full set of order-independent aggregates (COUNT/MIN/MAX/SUM/AVG/STDDEV*/VAR*). Cosmetic rename `COUNT_DISTINCT` → `BUILDVALUE_OPT` reflects the broadened scope.
- **Highest-value findings**:
  - **New file-static helper** `is_buildvalue_opt_supported_function(FUNC_CODE)` in `px_heap_scan_checker.cpp` — returns true for 12 aggregate function codes. Replaces hardcoded 2-function whitelist.
  - **Two-heap dance** is the key engineering pattern: workers write accumulators to **heap 0** (process-wide, survives worker teardown); main-thread merge happens in heap 0; main-thread `read` then re-clones values into the calling thread's private heap so downstream `qexec_end_buildvalueblock_iterations`'s `pr_clear_value` finds them in the right heap. Without this dance, cleanup would either leak (heap-0 not freed via worker heap) or crash (wrong-heap free).
  - **`qdata_aggregate_accumulator_to_accumulator`** is the standard CUBRID merge primitive (used by hash GROUP BY and serial aggregation) — reused here for the per-worker partial-aggregate merge. Saves duplicating per-aggregate merge logic.
  - **MIN/MAX(DISTINCT) shortcut**: DISTINCT is a no-op for MIN/MAX (extrema don't care about duplicates). The DISTINCT path has explicit `agg_node->function != PT_MIN && agg_node->function != PT_MAX` guards in 5 places (write_initialize, write, write_finalize, read).
  - **STDDEV/VAR uses two accumulator slots**: `value` (sum of x), `value2` (sum of x²). Both incrementally updated per row; final stddev/variance computation deferred to merge-time `qdata_aggregate_accumulator_to_accumulator`.
  - **Worker `write_initialize` failure propagation**: alloc/qfile_open failures now set `ER_OUT_OF_VIRTUAL_MEMORY` + `ERROR_INTERRUPTED_FROM_WORKER_THREAD` interrupt code via `move_top_error_message_to_this`. New 4-line `er_errid()` check after `write_initialize` in `task.cpp` consumes this.
  - **`agg_domains_resolved = 0`** forced at top of `write_initialize` so each worker re-resolves domains fresh per scan. Previous state could leak from prior scan.
  - **First-row vs Nth-row pattern** governed by `acc->curr_cnt < 1` — distinguishes "need to coerce + clone into accumulator domain" from "use `qdata_add_dbval` to incrementally update".
- **Incidental enhancements** (2): added `qdata_aggregate_accumulator_to_accumulator` documentation to result-handler; cross-link from aggregate-analytic to BUILDVALUE_OPT fast path.
- **Baseline impact**: triggered bump `175442fc` → `65d69154` (full hash `65d6915437eb6217ab0050939c6ad63f0d509735`). See `[2026-04-27] baseline-bump` entry above.

## [2026-04-27] manual-ingest | CUBRID 11.4 English User Manual — full catalog ingest
- Source: `/home/cubrid/cubrid-manual/en/` — Sphinx-RST documentation tree, 119 RST files, ~88,270 lines, ~37 MB. Manual version 11.4.0.1778 (matches baseline `175442fc`).
- **Strategy**: catalog + enhance (NOT a full per-file ingest — that would create 119 redundant pages). 21 cluster-source pages + 1 overview hub. Each source page is an outline + key facts + cross-refs back to the RST tree.
- **Parallel-agent dispatch** for the 4 large clusters (sql/, admin/, api/, pl/) — each agent returned a structured catalog (per-file 1-line summary, cluster strategy recommendation, top-15 enhancement opportunities, top-10 gap pages, 20-30 notable facts).
- **Pages created (22)**:
  - Hub: [[sources/cubrid-manual-en-overview]]
  - Top-level: [[sources/cubrid-manual-intro]], [[sources/cubrid-manual-install]], [[sources/cubrid-manual-csql]], [[sources/cubrid-manual-ha]], [[sources/cubrid-manual-shard]], [[sources/cubrid-manual-security]], [[sources/cubrid-manual-release-notes-114]]
  - Admin: [[sources/cubrid-manual-admin]], [[sources/cubrid-manual-config-params]], [[sources/cubrid-manual-error-codes]]
  - API: [[sources/cubrid-manual-api]], [[sources/cubrid-manual-cci]], [[sources/cubrid-manual-jdbc]]
  - PL: [[sources/cubrid-manual-pl]], [[sources/cubrid-manual-plcsql]]
  - SQL: [[sources/cubrid-manual-sql-foundation]], [[sources/cubrid-manual-sql-dml]], [[sources/cubrid-manual-sql-ddl]], [[sources/cubrid-manual-sql-tuning-parallel]], [[sources/cubrid-manual-sql-functions]]
- **Pages updated (incidental enhancements applied)**: [[components/cub-master]] (auto_restart_server NEW 11.4, HA split-brain detection, in-HA `cubrid server start` bypass trap), [[components/sp]] (pl_transaction_control, AUTHID Owner-only-for-PLCSQL, 16-call-via-query recursion limit, OUT-arg restriction, no-overloading, default-arg storage, DBMS_OUTPUT-only-package, JNI loadjava -j NEW 11.4), [[components/lock-manager]] (BU_LOCK introduced 10.2 for loaddb, 9 lock modes, lock_timeout -1/0 semantics, error code split table), [[components/btree]] (online index 3-stage protocol, DEDUPLICATE 0-14, ONLINE PARALLEL ignored in SA), [[components/optimizer]] (28-hint catalog table, plan cache regen rule 6 min + 10× page change, 11.4 stat improvements), [[components/error-manager]] (full namespace partitioning: server / CAS -10K / CCI -20K / JDBC -21K), [[components/recovery]] (-1128/-1129 NOTIFICATIONs, parallel REDO 11.4, HA force_remove_log_archives must be no).
- **Deferred enhancements**: each `cubrid-manual-*` page lists ~3-15 incidental enhancement opportunities in its "Incidental Wiki Enhancements" section — top-7 highest-impact applied this session, ~40+ remaining documented in source pages but not yet applied to component pages.
- **Key insight**: the manual ⇄ source split is the natural boundary — end-user contracts on one side, implementation on the other. Anchoring both to the same baseline commit (`175442fc`) lets us reconcile drift mechanically. **CUBRID 11.4 marquee features**: PL/CSQL (Oracle PL/SQL-compatible language), HASH JOIN (USE_HASH hint), LEADING hint, expanded query cache (CTE + uncorrelated), parallel REDO recovery, `cubrid memmon`, `auto_restart_server` for non-HA, `loadjava -j` for JNI Java SP, `restore_to_newdb.sh`. **Breaking specs**: CHAR max chars 268M → 2048, LOB locator path absolute → relative, AUTO_INCREMENT+DEFAULT illegal, `db_serial.att_name → attr_name`, view creation type checks deferred.
- **Manifest**: `.raw/.manifest.json` updated with tree-hash `85c1ebbc75933acda015a8776506a270` over the concatenated sorted .rst files.

## [2026-04-26] pr-ingest-deep | PR #7011 — Support parallel index build (OPEN, 9 files)
- Ingested CUBRID upstream [PR #7011](https://github.com/CUBRID/cubrid/pull/7011) "[CBRD-26678] Support parallel index build" by `@xmilex-git` (the user themselves). State `OPEN` (non-draft, 5 approvals). Base `develop@66e9279003`, head `parallel_index_build@44d92db64`. 9 files, +1058/−99 (1157 LOC). Largest: btree_load.c (+468/−46), external_sort.c (+425/−46). 1 file > 500 LOC.
- **Scale rule triggered** (1 file > 500 changed lines). Dispatched 2 parallel deep-read subagents — (A) btree_load.c/.h covering `SORT_ARGS` migration + per-thread getters + sysop layout, (B) external_sort.c/.h covering `SORT_INDEX_LEAF` wiring + `sort_merge_run_for_parallel_index_leaf_build` + tree-merge fan-in.
- **PR page**: [[prs/PR-7011-parallel-index-build]] with full Reconciliation Plan for 7 component pages + 1 candidate ADR. Most greptile P0/P1 alerts addressed at PR head.
- **Goal**: parallelize CREATE INDEX heap-scan + sort phase by reusing existing parallel-sort infrastructure. Adds wiring for the previously-declared-but-unused `SORT_INDEX_LEAF` enum value.
- **Highest-value findings**:
  - **Resolves a previously-flagged baseline gap**: `sort_copy_sort_param` implementation now provided at `external_sort.c:4344-4471` (the wiki's [[hot|hot cache]] noted this declaration was dangling; updated to point at PR #7011 + correct location).
  - **Class `parallel_heap_scan::ftab_set` generalized to `parallel_query::ftab_set`**: file moved from `src/query/parallel/px_heap_scan/px_heap_scan_ftab_set.hpp` → `src/query/parallel/px_ftab_set.hpp`. Old namespace preserved as `using` alias. Class gains move/copy semantics, `append()`, `move_from()`, `size()`.
  - **`SORT_ARGS` promoted from `static` (in btree_load.c) to public** (in btree_load.h). New fields: `filter_index_info`, `ftab_sets`, `curr_sec`, `curr_pgoffset`. New public type `FILTER_INDEX_INFO`. 5 functions promoted from `static` to `extern` (`bt_load_heap_scancache_*`, `bt_load_clear_pred_and_unpack`, `btree_load_filter_pred_function_info`, `btree_sort_get_next_parallel`).
  - **Per-thread XASL filter/func-index re-deserialization** required because `PRED_EXPR_WITH_CONTEXT::cache_pred` is mutated during evaluation; sharing one unpacked predicate across N workers would race the cache.
  - **Sysop ownership delegation**: SERVER_MODE delegates `log_sysop_start` + `btree_create_file` + `vacuum_log_add_dropped_file` to the sort layer (parallel: inside `sort_merge_run_for_parallel_index_leaf_build` AFTER merge fan-in completes; single-process: inside `sort_listfile` before `sort_listfile_internal`). SA_MODE retains existing serial flow with btree_load.c-side setup. Caller `xbtree_load_index` sets `is_sysop_started=true` under `#if defined(SERVER_MODE)` guard so it can abort/commit later.
  - **`btree_create_file` hoisted to merge phase** for SERVER_MODE parallel — shortens sysop life and means worker-fail never produces a partial btree file.
  - **Tree-merge fan-in**: `SORT_PX_MERGE_FILES = 4`, depth = `ceil(log₄ parallel_num)`. Empty (0-page) workers skipped during merge input-gathering.
- **Latent bugs the author fixed during review** (all addressed at PR head):
  - `pgbuf_ordered_unfix` missing on early-exit paths in `get_next_vpid` (commit `ab8ca3a`) — real page-pin leak.
  - `is_sysop_started` set twice in SERVER_MODE / missing in SA_MODE (commit `42b92007`).
  - `new`-allocated `ftab_sets` freed via `free_and_init` (UB): now `malloc + placement_new` paired with explicit `~vector() + free_and_init`.
  - SORT_ARGS double-free hazard on partial malloc loop: pre-NULL fix.
  - `sort_listfile_execute` cleanup missing for SORT_INDEX_LEAF: scancache cleanup added at line 1517.
  - Get_fn placement hoisted into one-shot setup loop (commit `e71b21b`).
  - `sort_put_result_for_parallel` dead-code reverted (commit `e71b21b`).
- **Silent fixes the PR drops in** (not in body): `external_sort.c:1491` `if (sort_param == NULL)` → `if (px_sort_param == NULL)` (was always-true on wrong pointer); `external_sort.c:1102` `malloc` → `calloc` for `file_contents[j].num_pages` (avoids uninit read on empty-worker runs).
- **Residual smells** beyond bot review (not bugs at head):
  - SA_MODE / SERVER_MODE single-process asymmetry — different ownership of scancache + sysop + btree_create_file.
  - `btree_sort_get_next_parallel` declared in `external_sort.h` instead of `btree_load.h` (cross-module).
  - `sort_merge_run_for_parallel_index_leaf_build` lacks `static` despite static forward decl.
  - `pow()` for integer index math — author defended ("원래 코드가 그래", safe in practice).
  - `std::vector` in btree_load.h public header — wrapped in `*INDENT-OFF*` comments suggesting awareness, but if any C TU pulls the header it fails. Verify all includers are C++.
  - Multi-class parallelism gated to `n_classes == 1` — plumbing supports it, but `sort_check_parallelism` short-circuits.
- **Incidental wiki enhancement**: [[hot]] cache entry annotated to mark `sort_copy_sort_param` baseline gap as resolved by PR #7011.
- **Baseline**: NOT bumped (PR is open).

## [2026-04-26] pr-ingest-deep | PR #6443 — System Catalog refactor for Information Schema (MERGED, case-b, 86 files)
- Ingested CUBRID upstream [PR #6443](https://github.com/CUBRID/cubrid/pull/6443) "[CBRD-25862] Improve and refactor System Catalog for Information Schema" by `@kangmin5505`. Merged 2025-12-12 (`1f12632170…`); ancestor of baseline `175442fc858b…` → **case (b) absorbed**, retroactive doc only, no bump.
- **Scale rule triggered on multiple axes**: 86 files, +2755/−1527 (4282 LOC); largest files `catalog_class.c` (+488/−114), `schema_system_catalog_install.cpp` (+468/−408), `class_object.c` (+167/−111), `schema_system_catalog_install_query_spec.cpp` (+165/−53). Dispatched 5 parallel deep-read subagents covering: (A) catalog disk format + storage, (B) install + view query-spec, (C) in-memory + schema + trigger refactor, (D) auth + user catalog (db_authorizations removal), (E) SP/PL/serial/index/executor/boot.
- **PR is a rollup over the long-lived `feature/system-catalog` branch** with 14+ sub-commits including `f4f2d857c` (db_authorizations removal #6039), `09128e0e9` (naming convention), `b6b210dc3` (start_val), `8c063d7b8` (is_loginable/is_system_created), `789378dfa` (sql_data_access), `c1b0801fe` (class_partition_type), `2a61602a1` (_db_index attrs), `971caa5bf` (statistics_strategy), `9f973f9e1` (revert is_system_class rename + add flags), `0b00be457` (checked_time NULL init).
- **PR page**: [[prs/PR-6443-system-catalog-information-schema]] with full Deep-analysis section consolidating 38 supplementary findings.
- **Highest-value findings** (most NOT in any prior review):
  - **No migration path for any of the new catalog columns.** `migrate.c` was not extended; `catcls_cache_fixed_attr_indexes` (`catalog_class.c:4691`) has hard `assert(_gv_… != -1)` for the new column indexes. **A server boot against an old database lacking these columns will SIGABRT.**
  - **Login fails on un-migrated DBs.** `perform_login` calls `is_loginable_user()` which calls `obj_get(user, "is_loginable", &value)`; on an old DB lacking the column, returns false silently → every login attempt fails with `ER_AU_LOGIN_DISABLED`.
  - **`unloaddb` does NOT include `start_val` in its `_db_serial` SELECT** (`unload_schema.c:704-711`). Combined with the missing migration, a round-trip `unloaddb`/`loaddb` cycle silently zeros every user-supplied `start_val` to default `1`.
  - **`register_user_trigger` and `unregister_user_trigger` have INVERTED if-error logic** (`trigger_manager.c:1968-1971, 2036-2039`): user-timestamp update fires only on FAILURE, AND overwrites the original error code with the timestamp helper's return. Silently breaks user-cache timestamping AND swallows real errors. Same defect duplicated in both functions.
  - **`smt_add_constraint_to_property` has uninitialized `current_datetime`** (`schema_template.c:1550`) that gets passed to `pr_clear_value` on the early-goto path → undefined behavior.
  - **Wire format extended without protocol-version handshake** — `compile_response` and `sql_semantics` packers gained trailing `dependencies` field; old C/Java pair will misalign.
  - **`flags` column split is catalog-row-only.** The on-disk heap-class record's single `flags` int (offset `ORC_CLASS_FLAGS = 64`) is unchanged. `catalog_class.c:1055-1060` splits at the catalog-row writer: bit 0 → `_db_class.is_system_class`, bits 1+ → `_db_class.flags`. Comment claims "recombine on write for compatibility" — recombine code does not exist (works because writes come from heap transform).
  - **`SP_SQL_DATA_ACCESS_TYPE` written as garbage on every `CREATE PROCEDURE`** — no parser path sets `sp_info::sql_data_access`; default `UNKNOWN = -1` is what gets written. Reserved-but-broken column. View doesn't expose it (commented-out TODO).
  - **`db_partition.class_partition_type` view loses info for value 1** (DB_PARTITIONED_CLASS) — CASE only emits `'PARTITION CLASS'` for value 2. Root partitioned tables look identical to non-partitioned tables.
  - **`db_authorizations` removed entirely** (sub-commit `f4f2d857c`, May 2025). Methods-only "old root" class with `add_user`, `drop_user`, `find_user`, etc. — duplicating `db_root`. Pre-existing databases keep an orphaned `_db_class` row pointing to a non-existent vclass spec (no migration drops it).
  - **Authorization defense-in-depth lost.** Every catalog write that used to ride on the user's grant now `AU_DISABLE`s and re-implements auth checks in C (e.g. `do_drop_serial` calls `au_check_serial_authorization` ad hoc). New paths must remember the check; nothing enforces centrally.
  - **`disable_login()` is dead code** — declared private in `authenticate_context.hpp:196`, defined at `authenticate_context.cpp:974-989`, **zero callers** tree-wide.
  - **`_db_stored_procedure_code.created_time` is `format_varchar(16)`** instead of datetime — inconsistent with every other `created_time` in the catalog. Likely SP-code-version-hash leftover.
  - **`SM_CLASSFLAG_SYSTEM` defined in two places** (`class_object.h:307` AND `catalog_class.c:67-68` with comment "Keep in sync") — two-place truth invites drift.
  - **`change_serial_owner` CLASS_METHOD still registered** (`schema_system_catalog_install.cpp:1014`) despite sub-PR claim of removal. Either rename was the actual scope or removal is incomplete.
  - **`_db_charset.charset_id` not indexed** despite being a join target for `db_collation`/`db_charset` views — inconsistent gap relative to the new `_db_collation.coll_id` PK.
  - **PR description is misleading on `is_system_class`** — claims `flags` "replaces" it; actual code keeps both columns.
  - **`network_interface_cl.c -214 LOC is NOT auth-related**: it rewrites `stats_update_all_statistics` from a hand-rolled SQL union to `locator_get_all_class_mops + is_top_level_class` filter loop.
- **Incidental wiki enhancement** applied (baseline-only facts):
  - [[components/system-catalog]] — major update: new sections "Naming Convention" (with the `db_user`/`db_root` exemptions and `db_authorizations` removal note), "Catalog Row Provenance" (per-row time columns, `flags` split rule, `catcls_cache_fixed_attr_indexes` SIGABRT risk), "Position-Index Enums" (`CT_ATTR_CLASS_INDEX`/`CT_ATTR_INDEX_INDEX` from transform.h), "Catalog Performance Indexes" (the three new indexes with rationale, plus the warning about `_db_class.class_of` non-uniqueness).
- **Baseline**: NOT bumped (case b).

## [2026-04-25] source-ingest | `src/optimizer/rewriter/` — per-file ingest of all 8 files
- Per-file ingest of `~/dev/cubrid/src/optimizer/rewriter/` at baseline `175442fc`. 8 files, 10,212 LOC total. Two files crossed the Scale rule threshold (>3000 LOC each): `query_rewrite_select.c` (3795 LOC) and `query_rewrite_term.c` (4294 LOC). Dispatched 2 parallel deep-read subagents in a single message for those two; read the smaller files (orchestrator + header + set + subquery + auto-parameterize + unused-function) directly in the main thread.
- **New wiki pages** (7): [[components/optimizer-rewriter]] (parent + orchestrator covering `query_rewrite.c/.h`), [[components/optimizer-rewriter-select]], [[components/optimizer-rewriter-term]], [[components/optimizer-rewriter-subquery]], [[components/optimizer-rewriter-set]], [[components/optimizer-rewriter-auto-parameterize]], [[components/optimizer-rewriter-unused-function]] (dead code, gated `#if defined(ENABLE_UNUSED_FUNCTION)`).
- **Components index updated**: added a new `## Optimizer (src/optimizer/)` section between Parser and Threading, listing `[[components/optimizer]]` plus the 7 new rewriter pages.
- **Key facts surfaced**:
  - 3-phase orchestrator pipeline in `mq_rewrite`: pre-rewrite (statement-shape-specific) → optimization (CNF + reduce_equality + 6-stage `qo_rewrite_terms` + `qo_rewrite_select_queries`) → auto-parameterize (constant → host-var marker for XASL-cache reuse, gated by 5 conditions including `!hostvar_late_binding` and `xasl_cache_max_entries > 0`).
  - `qo_rewrite_terms` 6-stage pipeline: converse-sarg → comp-pair → LIKE-rewrite → range-conversion → range-intersection → IS-NULL-fold. CNF (`->next`) × DNF (`->or_next`) shape assumed throughout.
  - `PT_BETWEEN_*` 9 sub-ops fully documented (`GE_LE`, `GE_LT`, `GT_LE`, `GT_LT`, `EQ_NA`, `INF_LE`, `INF_LT`, `GE_INF`, `GT_INF`).
  - `comp_dbvalue_with_optye_result` enum (note typo: `optye` should be `optype`) with `Adj` adjacency semantics critical for `(a > 5) OR (a = 5)` → `(a >= 5)` collapse.
  - **Outer-join correctness invariant**: `info.expr.location > 0` tags ON-clause terms; orchestrator promotes them to WHERE for unified rewriting, post-walk splices them back to the right spec's `on_cond` based on location match. Mutating a spec's location requires `qo_reset_spec_location`.
  - `qo_reduce_equality_terms` does derived-table column flattening + transitive-join inference (cloning equality terms into join terms with `PT_EXPR_INFO_TRANSITIVE` flag, consumed downstream by planner cardinality estimation).
  - **HISTOGRAM/BUCKETS-style breaking-change-class observation NOT present** — the rewriter is a pure transformation layer, doesn't introduce reserved words.
  - **Multi-fix LIKE rewrite** with collation gate (`lang_get_collation->options.allow_like_rewrite`) and synthetic `PT_LIKE_LOWER_BOUND`/`PT_LIKE_UPPER_BOUND` operators evaluated at scan time for index-scan eligibility.
  - **FK→PK parent-table elimination** with documented limitation: string-rendered constant comparison via `parser_print_tree(...PT_CONVERT_RANGE)` defeats locale-formatting differences (`1` vs `1.0`).
  - **32-column composite-key ceiling** baked into `cons_attr_flag = (1 << i) - 1` bitmap — undocumented.
  - **Right-outer not supported** in `qo_reduce_outer_joined_tbls` — TODO at :1921.
  - 5+ TODOs in LIKE rewriting (`PT_NOT_LIKE` not optimized, escape-elimination missing, "should check column is indexed" missing).
  - Header `query_rewrite.h:20` carries the comment "don't include this except files in this folder" — the rewriter exposes only `mq_rewrite` to the outside world.
- **Cross-file API consumed externally**: `qo_check_nullable_expr` (the only rewriter symbol called outside `optimizer/parser/` directories — by `optimizer/query_planner.c:11037` and `parser/parser_support.c:3773`).

## [2026-04-24] pr-ingest-deep | PR #6689 — BCB mutex → atomic_latch (MERGED, case-b via sibling PR #6704)
- Ingested CUBRID upstream [PR #6689](https://github.com/CUBRID/cubrid/pull/6689) "[CBRD-26425] Replace bcb mutex lock into atomic_latch" by `@xmilex-git`. State `MERGED` into feature branch `feature/refactor-pgbuf` on 2025-12-11 at commit `dedd387e6` — NOT an ancestor of baseline. Classified **case (d) by hash, case (b) in substance**: the equivalent changes landed in baseline via sibling re-merge PR #6704 (`58cef8e01`), so the baseline `page_buffer.c` already contains 83 `atomic_latch` references and the `pgbuf_thread_variables_init` prototype at `page_buffer.h:481`. Retroactive doc only — no reconciliation plan, no baseline bump. Rate-limit-delayed ingest (cron-scheduled 2h43m earlier in the day).
- **Scale rule triggered**: 8 files, +950/−502 LOC, but `page_buffer.c` alone is +932/−497 (>500 threshold). Dispatched 2 parallel subagents in a single message: (A) `page_buffer.c` + `page_buffer.h` core CAS refactor, (B) `thread_entry.hpp/.cpp/_task.cpp` + 3 non-pgbuf callers.
- **PR page**: [[prs/PR-6689-bcb-mutex-to-atomic-latch]] with "Deep analysis — supplementary findings" consolidating 17 code-only observations (the PR has **zero reviewer discussion** — no reviews, no inline comments, only author `/run all` triggers).
- **Highest-value new findings** (code-only; all untriaged):
  - **5 latent correctness bugs**: (a) `pgbuf_wakeup_reader_writer:7249` potential double-lock on CAS retry — `thread_lock_entry` inside the retry loop; (b) `pgbuf_unlatch_bcb_upon_unfix:6449-6463` detects `fcnt < 0` via assert but never CAS-clamps the atomic back to 0, leaving latch state corrupt in release builds; (c) `pgbuf_bcb_register_fix` bypassed on the lockfree RO fix path — hot-page detection silently skewed to contended pages only; (d) `pgbuf_assign_private_lru`/`release_private_lru` mutate `private_lru_index` without re-running `pgbuf_thread_variables_init` (needs code verification); (e) `recycle_context` asymmetry — resets `private_lru_index = -1` but not `m_is_private_lru_enabled`.
  - **Baseline drift vs PR head** (attributable to PR #6704, not #6689): VPID recheck inside `pgbuf_lockfree_fix_ro` gate CAS (ABA defense), `OLD_PAGE_MAYBE_DEALLOCATED` accepted in the lockfree RO fast path, `latch_last_thread` widened from `SERVER_MODE && !NDEBUG` to unconditional `SERVER_MODE` (stale comment at `page_buffer.c:520` still claims NDEBUG gating).
  - **Dead code**: `copy_bcb` (`page_buffer.c:1282-1308`) is never called anywhere in baseline.
  - **Design smells**: 3 disparate init sites for `pgbuf_thread_variables_init` is fragile (encapsulated setter on `cubthread::entry` would enforce the invariant); `pgbuf_simple_fix`/`pgbuf_simple_unfix` still take BCB mutex around pure atomic operations — either vestigial or correctness gap.
- **Incidental wiki enhancements** applied (baseline facts, orthogonal to PR):
  - [[components/page-buffer]] — replaced stale `mutex: pthread_mutex_t` BCB field row with the new `atomic_latch: std::atomic<uint64_t>` packed-union row; added `latch_last_thread` diagnostic row.
  - [[components/page-buffer]] — new "Atomic latch model" section covering CAS primitives, lockfree RO fast path (`pgbuf_lockfree_fix_ro` + `pgbuf_search_hash_chain_no_bcb_lock` + `pgbuf_lockfree_unfix_ro`), and the `pgbuf_thread_variables_init` contract with `private_lru_index` writers (3 init sites + 4 lazy-init guards for daemons).
- **Baseline**: NOT bumped (case b).

## [2026-04-24] pr-ingest-deep | PR #6753 — Histogram optimizer support (OPEN, 64 files, +5335/−145)
- Ingested CUBRID upstream [PR #6753](https://github.com/CUBRID/cubrid/pull/6753) "[CBRD-26202] Add Optimizer Histogram Support" by `@sohee-dgist`. State `OPEN` (non-draft, 6 approvals), base `develop@1da6caa7` = wiki baseline, head `CUBRID-HISTOGRAM@9970b72a`.
- **Scale rule triggered**: 64 files × 5480 LOC total, largest file `histogram_cl.cpp` 1882 LOC. Dispatched 5 parallel subagents in a single message: (A) `src/optimizer/histogram/*` core (~3300 LOC across 6 new files), (B) optimizer integration (`query_planner.c/h`, `query_graph.c/h`, `statistics.h`), (C) parser/lexer/semantic/DDL (8 files), (D) object+catalog layer (23 files including `_db_histogram` install, `make_template is_read_only`, `obj_find_multi_attr` rewrite, ABI-break on `dbt_create_object_internal`), (E) executor/storage/sampling/CMake/misc (23 files including `do_*_histogram`, Poisson RNG, sampling weight formula).
- **PR page**: [[prs/PR-6753-optimizer-histogram-support]] written with full Reconciliation Plan (20+ target pages) and a "Deep analysis — supplementary findings" section consolidating 47 observations not covered by existing bot/reviewer threads.
- **Highest-value new findings** (not in any prior review):
  - `HISTOGRAM` and `BUCKETS` become **fully-reserved words** (no `identifier:` fallback). Any existing CUBRID schema using either as a column or table name fails to parse post-merge. Fix pattern: follow `HEAP`/`FULLSCAN`.
  - `oid_Histogram_class` cache slot declared and enumerated (`OID_CACHE_HISTOGRAM_CLASS_ID`) but **never populated** by `boot_client_find_and_cache_class_oids`. The new MVCC-snapshot-skip in `btree.c:24783` for the histogram catalog therefore never fires — the intended isolation behavior is disabled.
  - `update_histogram_for_all_classes` leaks `LIST_MOPS` on every call and can bypass `AU_ENABLE` on the `lmops == NULL` early-return.
  - **Privilege gap**: all three histogram DDL wrappers (`do_update_histogram`, `do_drop_histogram`, `do_show_histogram`) `AU_DISABLE`-wrap their full body. No ownership check → any user with `SELECT` on a table can `ANALYZE TABLE t DROP HISTOGRAM` on it.
  - **TRUNCATE does NOT invalidate histograms** → stale buckets reference deleted rows; planner is silently wrong on truncate-and-reload workloads.
  - **Unload regression**: `_db_histogram` added to `unload_object.c` prohibited_classes; `cubrid unloaddb` silently loses histograms. View TODO still commented-out.
  - **Default bucket count is 300**, not 30 (early review thread is stale — `default_histogram_bucket_count` sysprm, default=min=300, max=1000).
  - **Blob format doc contradicts code**: header comment says magic `HST2`, endian `LE`, `f64 cumulative` — actual is `HST1`, big-endian (via `OR_PUT_INT`), `int64_t`.
  - **ABI break**: `dbt_create_object_internal(DB_OBJECT *)` → `dbt_create_object_internal(DB_OBJECT *, bool is_read_only)`. Out-of-tree consumers of `libcubridcs`/`libcubridsa` won't link. In-tree drivers rebuild fine.
  - **`PT_SHOW_HISTOGRAM` is mistakenly included** in `do_replicate_statement` — read-only statement should never replicate. Slave's `la_apply_statement_log` correctly ignores on replay, but bandwidth is wasted.
- **Incidental wiki enhancements applied** (baseline-only facts, not PR-induced):
  - [[components/optimizer]] — added "Selectivity defaults" section listing the 9 `DEFAULT_*_SELECTIVITY` constants and `PRED_CLASS` enum (previously absent).
  - [[components/scan-manager]] — added "Sampling Scan" sub-section covering `S_HEAP_SAMPLING_SCAN`, weight formula, `stats_adjust_sampling_weight`.
  - [[components/heap-file]] — added "Sampling Scan Integration" note about `heap_next_internal` page-stride sampling.
  - [[components/system-catalog]] — documented the `CNT_CATCLS_OBJECTS` invariant and its role in `schema_class_truncator.cpp`.
  - [[components/object]] — added "Index-Value Writers" note explaining `mr_index_writeval_oid`'s dual acceptance of `DB_TYPE_OBJECT` and `DB_TYPE_OID`.
- **Baseline**: NOT bumped (PR is open). Reconciliation Plan in the PR page is executable on merge via `apply reconciliation for PR #6753`.

## [2026-04-24] pr-ingest-deep | PR #7062 — Deep-read pass with 5 parallel subagents
- Expanded the initial [[prs/PR-7062-parallel-scan-all-types|PR-7062]] ingest beyond the authoritative external review doc by dispatching 5 parallel sub-agents to read the full 10,396-line diff line-by-line across file clusters: (1) `px_scan.cpp` + `task`, (2) `slot_iterator_index`, (3) input handlers (list/index/heap + ftab_set + slot_iterator_list), (4) `result_handler` + `checker` + `trace_handler`, (5) C-side integration (`scan_manager`, `query_executor`, `plan_generation`, grammar/parser, `btree.c/h`, `system_parameter`).
- **Key corrections surfaced** (added as "Deep analysis — supplementary findings" section in the PR page):
  - **Two system parameters, not one.** `PRM_ID_PARALLEL_SCAN_PAGE_THRESHOLD` (server-hidden, default 2048) is the runtime kill-switch; `PRM_ID_PARALLEL_INDEX_SCAN_PAGE_THRESHOLD` (client-session, default 32) is a separate optimizer tuning knob used by the new cost logic in `plan_generation.c`. The review doc documented the merged history, not the final state.
  - **BUILDVALUE_OPT scope far beyond COUNT DISTINCT.** Rename from `COUNT_DISTINCT` → `BUILDVALUE_OPT` reflects an actual capability expansion — whitelist of 11 aggregates (`COUNT_STAR`, `COUNT`, `MIN`, `MAX`, `SUM`, `AVG`, `STDDEV*`, `VARIANCE*`, `GROUP_CONCAT`) via new `is_buildvalue_opt_supported_function()`.
  - **Deferred-promotion pattern for index scan.** `scan_open_parallel_index_scan` stashes `parallel_pending` in the union; `scan_start_scan` → `scan_try_promote_parallel_index_scan` consumes it. Heap and list have no analogous state. `qexec_intprt_fnc` clears the pending when `need_count_only` flips.
  - **`qexec_resolve_domains_for_aggregation_for_parallel_heap_scan_buildvalue_proc` retains `heap_scan` in its name post-merge** — latent rename inconsistency.
  - **New public btree helpers.** `btree_leaf_record_is_fence` (new), `btree_key_process_objects` + `BTREE_PROCESS_OBJECT_FUNCTION` (moved from file-static to public). `scan_regu_key_to_index_key` promoted from static to extern.
  - New 137-line optimizer cost function `qo_apply_parallel_index_scan_threshold` in `plan_generation.c` (computes `metric = ceil(sel * index_leaf_pages)`, degree `floor(log2(metric/threshold)) + 2`).
  - 8 dispatch points in `scan_manager.c` receive `S_PARALLEL_LIST_SCAN` + `S_PARALLEL_INDEX_SCAN` cases. `qexec_clear_access_spec_list` consolidated from inline switch to a single wrapper.
- **Incidental wiki enhancements expanded from 1 → 4** (all applied now as baseline-truth additions):
  - [[components/xasl]] (from first pass) — ACCESS_SPEC_FLAG_* 7-flag table.
  - [[components/btree]] — fence-key contract, BTREE_NODE_HEADER leaf-chain fields, non-leaf record 6-byte prefix layout, MVCC visibility filtering via `btree_mvcc_info_to_heap_mvcc_header`, `btree_key_process_objects` public API.
  - [[components/list-file]] — QFILE_LIST_ID `dependent_list_id` chain, membuf vs disk backing, `QFILE_OVERFLOW_TUPLE_COUNT_FLAG = -2` overflow sentinel.
  - [[components/file-manager]] — `file_get_all_data_sectors` / `FILE_FTAB_COLLECTOR` API under Layer 1.
- Code-concerns backlog added to PR page: dangling `heap_scan` in function name, surviving `TODO: 0 ?` in system_parameter.c, redundant reset_scan_block field clears, generic `ER_FAILED + er_clear()` in index fallback, missing leaf-level assert after `pgbuf_fix`, deferred key-range conversion in `set_page`, missing NULL guard on `indx_info->key_info.key_ranges`, hidden non-leaf byte-layout contract.
- Lesson: initial pass did file-sampling based on the review doc + skimmed diff; user pushback required dispatching 5 parallel subagents for full-read coverage. Incorporated into protocol — see next log entry.

## [2026-04-24] pr-ingest | PR #7062 — Expand parallel heap scan to parallel scan (index, heap, list) (CBRD-26722)
- Upstream: https://github.com/CUBRID/cubrid/pull/7062 · state **OPEN** (non-draft) · author `@xmilex-git`
- Case: **open** — Reconciliation Plan written in full, no component-page edits applied for PR-induced changes. `reconciliation_applied: false`. Executable later via "apply reconciliation for PR #7062".
- Filed: [[prs/PR-7062-parallel-scan-all-types|PR-7062]] (first invocation of the new state-aware PR ingest protocol).
- Scope: 52 files, ≈ +6,190/−1,893 LOC, 77 commits. Retires `src/query/parallel/px_heap_scan/` namespace in favour of generalised `src/query/parallel/px_scan/`; adds `input_handler_{list,index}` + `slot_iterator_{list,index}`; `manager<RT>` becomes `manager<RT, ST>` over `SCAN_TYPE::{HEAP,LIST,INDEX}`.
- Key design: mutex-guarded leaf-chain cursor for index scans (amortised via `parallel_scan_page_threshold ≥ 2048`); sector pre-split + membuf fallback for list scans; heap path unchanged (rename only). Result matrix is 7-of-9 (XASL_SNAPSHOT × {LIST,INDEX} blocked in checker).
- Author-attached external design doc: https://github.com/user-attachments/files/26920618/pr_7062_code_review.md (358 lines — primary source for the PR page synthesis).
- Reconciliation Plan covers 16 existing pages (parallel-heap-scan family rename + semantic repurposing, scan-manager enum/union additions, checker expansion, xasl flag renames, btree integration, parser rename propagation) + 5 new pages to create on merge (`parallel-scan-input-handler-{list,index}`, `parallel-scan-slot-iterator-{list,index}`, `parallel-scan-type`).
- Initial pass incidental enhancement: [[components/xasl]] — added a complete `ACCESS_SPEC_FLAG_*` table (7 flags with bit values + semantics) + paragraph on the `PT_HINT_* → PT_SPEC_FLAG_* → ACCESS_SPEC_FLAG_*` propagation pipeline. Previously the page only mentioned 2 flags in prose.
- No baseline bump (PR not merged).
- Superseded by "pr-ingest-deep" entry above once the user requested full-read coverage.

## [2026-04-24] pr-ingest | PR #6911 — Reduce I/O bottleneck when parallel heap scan (CBRD-26615)
- Upstream: https://github.com/CUBRID/cubrid/pull/6911 · merge commit `45730b9` · merged 2026-03-27
- Case: (b) already absorbed — merge is 36 commits behind baseline `175442fc8`. Retroactive doc only; no wiki-page reconciliation, no baseline bump.
- Filed: [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR-6911]]
- Design summary: replaced per-page mutex handoff (`px_heap_scan_input_handler_single_table.cpp`) with upfront sector collection + worker-local bitmap walk (`px_heap_scan_input_handler_ftabs.cpp` + `file_get_all_data_sectors` in `file_manager.c`).
- Review-driven design convergence: initial attempt to widen `pgbuf_ordered_fix` with an `allow_not_ordered_page` bool was reverted after `@hornetmj`/`@shparkcubrid`/`@youngjinj` pushback — final merge has zero `page_buffer.c` churn. Candidate ADR noted but not filed.
- Related wiki pages (already post-PR state): [[components/parallel-heap-scan]], [[components/parallel-heap-scan-input-handler]], [[components/file-manager]], [[sources/cubrid-src-query-parallel]], [[sources/cubrid-src-storage]].

## [2026-04-24] lint + cleanup | Top-5 lint fixes + legacy seed archive
- Filed [[lint-report-2026-04-24]] (delta report vs 04-23).
- Applied top-5 lint fixes:
  1. `[[Wiki Map]]` → `[[Wiki Map.canvas|Wiki Map]]` across `index.md`, `overview.md`, `concepts/_index.md` (prior dead link on every vault open — resolved).
  2. `components/query-reevaluation` — cross-linked from `components/scan-manager` + `components/mvcc` (new orphan fixed).
  3. Frontmatter batch: `created: 2026-04-23` added to 46 pages missing the field; `status` added to 24 pages (10 `dependencies/*` → `reference`; 6 `parallel-*` components + 2 `flows/*` → `developing`; 6 `sources/*` → `active`).
  4. `decisions/_index.md` status: `active` → `stub` (directory still contains only `_index.md`).
  5. `overview.md` — removed 2 dead canvas wikilinks (`[[AI Marketing Hub Cover Images Canvas]]`, `[[claude-obsidian-presentation]]`).
- Legacy seed archived under `wiki/_legacy/`: 18 pre-CUBRID pages moved via `git mv` (history preserved):
  - 4 concepts (LLM Wiki Pattern, Hot Cache, Compounding Knowledge, cherry-picks)
  - 7 entities (Karpathy + 6 ecosystem-research projects)
  - 2 comparisons (whole dir)
  - 1 question (whole dir)
  - 3 plugin release-session meta pages
  - 1 ecosystem-research source
  - getting-started onboarding
- Created `[[_legacy/_index|Legacy Seed Index]]` as landing page.
- Hub pages rewritten: `index.md`, `overview.md`, `concepts/_index.md`, `entities/_index.md`, `meta/dashboard.md` — legacy first-class listings stripped, single pointer to `_legacy/_index` added.
- `CLAUDE.md` updated: secondary-scope paragraph now points to `_legacy/`; vault-structure block shows `wiki/_legacy/` as archived subdir; rule added: "do not extend — all new content goes into Mode B CUBRID structure".
- Dashboard dataview queries filtered out `_legacy` content; `comparisons/` + `questions/` queries removed (dirs moved).
- Net result: active-content dead-link count down to editorial backlog only (module/submodule stubs `cubrid-cci`/`cubrid-jdbc`/`cubridmanager` + CUBRID build-dir stubs `cs`/`sa`/`cubrid`/`conf`/`win`/`cm_common`). Zero unintentional orphans in active content.

## [2026-04-23] ingest | CUBRID round 5 — src/query + src/query/parallel DEEP DIVE (6 parallel agents)
- Scope: per-file / per-function granularity for all of `src/query/` top-level (80+ files) + `src/query/parallel/` (top-level + 3 subdirs)
- Agents dispatched: operators (7 pages), execution helpers (8 pages), scan families (4 + 1 upgrade), XASL+vacuum (6 upgrades/creates), parallel top-level (2 new + 4 upgrades), parallel subdirs (9 pages by agent + 5 manual after rate limit)
- Pages created/upgraded (total ~45): see [[components/_index]] "Query Operator Family", "Query Execution Layer", "Query Scan Types", "XASL / Cache", "Parallel Query" sections
- Key insights:
  - `arithmetic.c` owns 22 JSON scalar functions AND `SLEEP()` (server-thread usleep)
  - `DB_NUMERIC` is 16-byte two's-complement big-integer (NOT BCD as earlier pages suggested — corrected)
  - `string_opfunc.c` (~28K lines) has datetime sub-family (~8-10K) that arguably warrants its own file
  - Regex: RE2 default, std::regex `[[. .]]` collatename intentionally disabled (cross-compiler inconsistency)
  - DBLink password = time-seeded XOR obfuscation, not a cipher
  - `qdata_evaluate_generic_function` is a dead stub
  - AND short-circuits on V_FALSE but NOT on V_UNKNOWN (correct 3VL)
  - Hash join partition count computed upfront (no mid-build spill); Parallel gated on SERVER_MODE + non-Windows + xasl->parallelism > 1
  - scan-json-table re-evaluates RapidJSON Pointer per row (no path compile cache)
  - XASL cache: SHA-1 keyed, session prepared statement holds stale refs (intentional)
  - CAS reservation: `compare_exchange_weak` in-place update on failure; memory_order release/acquire pair
  - MPMC slot ABA: sequence cycles i → i+cap → i+2·cap + separate `ready` atomic bool
  - `atomic_instnum` uses `fetch_add` (CAS not needed — over-emit tolerated)
  - `err_messages::move_top_error_message_to_this()` SWAPS thread-local error (clears context)
  - REGISTER_WORKERPOOL runs at static-init (not in init()); call_once failure is permanent
- AGENTS.md inaccuracy: new SQL function registration = `qdata_evaluate_function` switch in `query_opfunc.c`, not just `fetch.c`
- Source-code defects surfaced: `sort_copy_sort_param` declared but not implemented in px_sort.c; `TASK_QUEUE_SIZE_PER_CORE = 2` constant unused; `reset_queue` epoch-bump invariant unclear

## [2026-04-23] ingest | CUBRID round 3e — `api`, `debugging`, `win_tools`, `heaplayers` (parallel)
- Sources: `.raw/cubrid/src/{api,debugging,win_tools,heaplayers}/`
- Summaries: [[cubrid-src-api]], [[cubrid-src-debugging]], [[cubrid-src-win-tools]], [[cubrid-src-heaplayers]]
- Pages created (5): [[components/api|api]], [[components/cubrid-log-cdc|cubrid-log-cdc]], [[components/debugging|debugging]], [[components/win-tools|win-tools]], [[components/heaplayers|heaplayers]]
- Key insights: CDC API bypasses broker/CAS (raw CSS connection to cub_server); `strict_warnings` listed in AGENTS.md but absent from tree (gap noted); Windows NT service uses SCM control codes 160-223 + registry-key sync-by-convention; `src/heaplayers/` is unmodified Emery Berger vendor copy (Apache 2.0), engine surface = `HL_HEAPID` opaque handle only.

## [2026-04-23] ingest | CUBRID round 3d — `executables`, `monitor`, `session`, `cm_common` (parallel)
- Sources: `.raw/cubrid/src/{executables,monitor,session,cm_common}/`
- Summaries: [[cubrid-src-executables]], [[cubrid-src-monitor]], [[cubrid-src-session]], [[cubrid-src-cm-common]]
- Pages created (12): [[components/executables|executables]], [[components/cub-server-main|cub-server-main]], [[components/csql-shell|csql-shell]], [[components/cub-master-main|cub-master-main]], [[components/utility-binaries|utility-binaries]], [[components/monitor|monitor]], [[components/perfmon|perfmon]], [[components/stats-collection|stats-collection]], [[components/session|session]], [[components/session-state|session-state]], [[components/session-variables|session-variables]], [[components/cm-common-src|cm-common-src]]
- Key insights: csql runtime `dlopen()` DSO lets one binary serve SA+CS without recompile; cub_master single-threaded `select()` loop with C++ `master_server_monitor` for auto-respawn; monitor stats always-on (no sampling), per-tran sheets reused without zeroing → snapshot delta required; session zero-hash hot path via `thread_p->conn_entry->session_p` pointer cache; `@vars` & session params NOT rolled back; each session owns its own page-buffer LRU zone.

## [2026-04-23] ingest | CUBRID round 3c — `broker(impl)`, `communication`, `method`, `loaddb` (parallel)
- Sources: `.raw/cubrid/src/{broker,communication,method,loaddb}/`
- Summaries: [[cubrid-src-broker]], [[cubrid-src-communication]], [[cubrid-src-method]], [[cubrid-src-loaddb]]
- Pages created (14): [[components/broker-impl|broker-impl]], [[components/cas|cas]], [[components/broker-shm|broker-shm]], [[components/shard-broker|shard-broker]], [[components/communication|communication]], [[components/packer|packer]], [[components/request-response|request-response]], [[components/method|method]], [[components/method-invoke-group|method-invoke-group]], [[components/method-scan|method-scan]], [[components/loaddb|loaddb]], [[components/loaddb-grammar|loaddb-grammar]], [[components/loaddb-executor|loaddb-executor]], [[components/loaddb-driver|loaddb-driver]]
- Key insights: broker ↔ CAS are separate OS processes (only IPC = 2 POSIX shm + socket fd via SCM_RIGHTS); 44 dispatch codes in CAS; shard `CON_STATUS_LOCK` uses POSIX sem on Linux, Peterson's algorithm on Windows; no server-server RPC layer (HA replication uses ordinary client-facing slots); `cubpacking::packer` shared with XASL stream; `method_invoke_group` struct lives in `src/sp/` but instantiated by method scanner (sp+method inseparable); loaddb has no parse tree (streaming model: grammar action → callback → `locator_multi_insert_force`).

## [2026-04-23] ingest | CUBRID round 3b — `compat`, `sp`, `thread`, `connection` (parallel)
- Sources: `.raw/cubrid/src/{compat,sp,thread,connection}/`
- Summaries: [[cubrid-src-compat]], [[cubrid-src-sp]], [[cubrid-src-thread]], [[cubrid-src-connection]]
- Pages created (19): [[components/compat|compat]], [[components/db-value|db-value]], [[components/client-api|client-api]], [[components/dbi-compat|dbi-compat]], [[components/sp|sp]], [[components/sp-jni-bridge|sp-jni-bridge]], [[components/sp-method-dispatch|sp-method-dispatch]], [[components/sp-protocol|sp-protocol]], [[components/thread|thread]], [[components/thread-manager|thread-manager]], [[components/worker-pool|worker-pool]], [[components/entry-task|entry-task]], [[components/thread-daemon|thread-daemon]], [[components/connection|connection]], [[components/cub-master|cub-master]], [[components/network-protocol|network-protocol]], [[components/heartbeat|heartbeat]], [[components/tcp-layer|tcp-layer]]
- Key insights: `DB_VALUE` is 3-field struct; `DB_TYPE` enum ABI-frozen on disk + XASL stream (new types append after `DB_TYPE_JSON=40` only); cub_pl is **separate OS process** (no in-process JNI), Unix domain socket + bidirectional callback loop; cub_master uses `SCM_RIGHTS sendmsg` for zero-copy fd handoff (out of data path after handshake); local clients auto-upgrade to Unix domain socket (no TCP overhead).

## [2026-04-23] ingest | CUBRID round 3a — `transaction`, `object`, `base`, `xasl` (parallel)
- Sources: `.raw/cubrid/src/{transaction,object,base,xasl}/`
- Summaries: [[cubrid-src-transaction]], [[cubrid-src-object]], [[cubrid-src-base]], [[cubrid-src-xasl]]
- Pages created (24): [[components/mvcc|mvcc]], [[components/lock-manager|lock-manager]], [[components/deadlock-detection|deadlock-detection]], [[components/log-manager|log-manager]], [[components/recovery|recovery]], [[components/vacuum|vacuum]], [[components/server-boot|server-boot]], [[components/object|object]], [[components/schema-manager|schema-manager]], [[components/system-catalog|system-catalog]], [[components/authenticate|authenticate]], [[components/lob-locator|lob-locator]], [[components/base|base]], [[components/error-manager|error-manager]], [[components/memory-alloc|memory-alloc]], [[components/lockfree|lockfree]], [[components/system-parameter|system-parameter]], [[components/porting|porting]], [[components/xasl|xasl]], [[components/xasl-stream|xasl-stream]], [[components/regu-variable|regu-variable]], [[components/xasl-predicate|xasl-predicate]], [[components/xasl-aggregate|xasl-aggregate]], [[components/xasl-analytic|xasl-analytic]]
- Pages updated: [[components/transaction]] (stub→comprehensive)
- Key insights: `wait_for_graph.c` is dead code (`ENABLE_UNUSED_FUNCTION` guard) — actual deadlock detection in `lock_manager.c` (CONTRADICTS AGENTS.md claim); vacuum physically lives in `src/query/` not `src/transaction/`; `authenticate_context` is C++ class, legacy `au_*` macros are shims (grep traps); `memory_wrapper.hpp` last-include is architectural (glibc placement-new conflict avoidance); lock-free ABA solved via epoch-based retirement (`lockfree::tran::system`); XASL serializes pointers as byte offsets, 256-bucket visited-pointer hashtable, UNPACK_SCALE=3 = server pre-allocates 3× stream size.

---

## [2026-04-23] ingest | CUBRID src/storage/ — Storage Layer
- Source: `.raw/cubrid/src/storage/` (57 files, AGENTS.md present)
- Summary: [[cubrid-src-storage]]
- Pages created: [[components/page-buffer|page-buffer]], [[components/btree|btree]], [[components/heap-file|heap-file]], [[components/file-manager|file-manager]], [[components/double-write-buffer|double-write-buffer]], [[components/overflow-file|overflow-file]], [[components/extendible-hash|extendible-hash]], [[components/external-sort|external-sort]], [[components/external-storage|external-storage]]
- Pages updated: [[components/storage|storage]] (stub → comprehensive), [[components/_index|components/_index]]
- Key insight: 3-zone LRU buffer pool; DWB recovery precedes WAL redo; WAL-ordering enforced inside `pgbuf_flush_with_wal`; LOB delete uses `LOG_POSTPONE`; B-tree dispatch is parameterized by 18 `btree_op_purpose` values.

## [2026-04-23] ingest | CUBRID src/parser/ — SQL Parser
- Source: `.raw/cubrid/src/parser/` (39 files, AGENTS.md present)
- Summary: [[cubrid-src-parser]]
- Pages created: [[components/parse-tree|parse-tree]], [[components/name-resolution|name-resolution]], [[components/semantic-check|semantic-check]], [[components/xasl-generation|xasl-generation]], [[components/view-transform|view-transform]], [[components/parser-allocator|parser-allocator]], [[components/show-meta|show-meta]]
- Pages updated: [[components/parser|parser]] (stub → comprehensive), [[components/_index|components/_index]]
- Key insight: PT_NODE function tables ordinal-indexed (silent crash on misorder); `YYMAXDEPTH 1000000` + `container_2..11` Bison helpers; `parser_block_allocator::dealloc` is no-op (arena lifetime); `mq_translate` runs `mq_reset_ids` per view inline.

## [2026-04-23] ingest | CUBRID src/query/ — XASL Execution Layer
- Source: `.raw/cubrid/src/query/` (84 top-level files, AGENTS.md present; parallel/ excluded — separate ingest)
- Summary: [[cubrid-src-query]]
- Pages created: [[components/query|query]], [[components/query-executor|query-executor]], [[components/scan-manager|scan-manager]], [[components/cursor|cursor]], [[components/partition-pruning|partition-pruning]], [[components/dblink|dblink]], [[components/list-file|list-file]], [[components/aggregate-analytic|aggregate-analytic]], [[components/filter-pred-cache|filter-pred-cache]], [[components/memoize|memoize]]
- Pages updated: [[components/_index]], [[sources/_index]]
- Key insight: `qexec_execute_mainblock` ~27K lines (intentional); `SCAN_ID` polymorphic over 15 scan types incl. PARALLEL_HEAP_SCAN/DBLINK/JSON_TABLE/METHOD; hash GROUP BY 2-phase spill (2000 tuple calibration, 50% selectivity); memoize self-disables after 1000 iters at <50% hit rate; partition pruning enables O(1) MIN/MAX on partitioned tables; filter_pred_cache exclusive lease (no shared locks).

## [2026-04-23] ingest | CUBRID src/query/parallel/ — Parallel Query Execution
- Source: `.raw/cubrid/src/query/parallel/` (16 files + 3 subdirs)
- Summary: [[cubrid-src-query-parallel]]
- Pages created: [[components/parallel-query|parallel-query]], [[components/parallel-worker-manager|parallel-worker-manager]], [[components/parallel-task-queue|parallel-task-queue]], [[components/parallel-hash-join|parallel-hash-join]], [[components/parallel-heap-scan|parallel-heap-scan]], [[components/parallel-query-execute|parallel-query-execute]], [[components/parallel-sort|parallel-sort]]
- Key insight: single global named pool ("parallel-query") with lock-free CAS reservation; logarithmic auto-degree (`floor(log2(pages/threshold))+2`); thread-local errors must be moved to shared `err_messages_with_lock`; spin-yield wait (not condvar) for short-lived bursts; SERVER_MODE/SA_MODE only.

## [2026-04-23] ingest | CUBRID AGENTS.md (project guide)
- Source: `.raw/cubrid/AGENTS.md` (md5 946ec27...)
- Summary: [[cubrid-AGENTS]]
- Pages created: [[CUBRID]], [[Query Processing Pipeline]], [[Build Modes (SERVER SA CS)]], [[Memory Management Conventions]], [[Error Handling Convention]], [[Code Style Conventions]], [[modules/src|src]], [[modules/broker|broker]], [[modules/pl_engine|pl_engine]], [[modules/unit_tests|unit_tests]], [[components/parser]], [[components/optimizer]], [[components/storage]], [[components/transaction]]
- Pages updated: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Key Decisions]], [[index]]
- Key insight: same source tree compiles into 3 binaries (`SERVER_MODE`/`SA_MODE`/`CS_MODE`); parser+optimizer run client-side; `.c` files are compiled as C++17.

## [2026-04-23] scaffold | CUBRID Mode B overlay
- Type: scaffold
- Mode: B (GitHub / codebase)
- Source tree: ~/dev/cubrid
- Created folders: wiki/modules, wiki/components, wiki/decisions, wiki/dependencies, wiki/flows
- Created hub pages: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Dependency Graph]], [[Key Decisions]]
- Created _templates: module, component, decision, dependency, flow
- Updated CLAUDE.md with CUBRID scope and Mode B conventions

## [2026-04-08] save | claude-obsidian v1.4 Release Session
- Type: session
- Location: wiki/meta/claude-obsidian-v1.4-release-session.md
- From: full release cycle covering v1.1 (URL/vision/delta tracking, 3 new skills), v1.4.0 (audit response, multi-agent compat, Bases dashboard, em dash scrub, security history rewrite), and v1.4.1 (plugin install command hotfix)
- Key lessons: plugin install is 2-step (marketplace add then install), allowed-tools is not valid frontmatter, Bases uses filters/views/formulas not Dataview syntax, hook context does not survive compaction, git filter-repo needs 2 passes for full scrub

## [2026-04-08] ingest | Claude + Obsidian Ecosystem Research
- Type: research ingest
- Source: `.raw/claude-obsidian-ecosystem-research.md`
- Queries: 6 parallel web searches + 12 repo deep-reads
- Pages created: [[claude-obsidian-ecosystem]], [[cherry-picks]], [[claude-obsidian-ecosystem-research]], [[Ar9av-obsidian-wiki]], [[Nexus-claudesidian-mcp]], [[ballred-obsidian-claude-pkm]], [[rvk7895-llm-knowledge-bases]], [[kepano-obsidian-skills]], [[Claudian-YishenTu]]
- Key finding: 16+ active Claude+Obsidian projects; 13 cherry-pick features identified for v1.3.0+
- Top gap confirmed: no delta tracking, no URL ingestion, no auto-commit

## [2026-04-07] session | Full Audit, System Setup & Plugin Installation
- Type: session
- Location: wiki/meta/full-audit-and-system-setup-session.md
- From: 12-area repo audit, 3 fixes, plugin installed to local system, folder renamed

## [2026-04-07] session | claude-obsidian v1.2.0 Release Session
- Type: session
- Location: wiki/meta/claude-obsidian-v1.2.0-release-session.md
- From: full build session — v1.2.0 plan execution, cosmic-brain→claude-obsidian rename, legal/security audit, branded GIFs, PDF install guide, dual GitHub repos


- Source: `.raw/` (first ingest)
- Pages updated: [[index]], [[log]], [[hot]], [[overview]]
- Key insight: The wiki pattern turns ephemeral AI chat into compounding knowledge — one user dropped token usage by 95%.

## [2026-04-07] setup | Vault initialized

- Plugin: claude-obsidian v1.1.0
- Structure: seed files + first ingest complete
- Skills: wiki, wiki-ingest, wiki-query, wiki-lint, save, autoresearch
