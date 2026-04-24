---
type: pr
pr_number: 6753
pr_url: "https://github.com/CUBRID/cubrid/pull/6753"
repo: "CUBRID/cubrid"
state: OPEN
is_draft: false
author: "sohee-dgist"
created_at:
merged_at:
closed_at:
merge_commit:
base_ref: "develop"
head_ref: "CUBRID-HISTOGRAM"
base_sha: "1da6caa7d6221f4543ef69a5deba12e074e4290b"
head_sha: "9970b72abe3e972aaf0e7741b2e9af862f7b0f5a"
jira: "CBRD-26202"
files_changed:
  - "CMakeLists.txt"
  - "cs/CMakeLists.txt"
  - "sa/CMakeLists.txt"
  - "src/base/ddl_log.c"
  - "src/base/system_parameter.c"
  - "src/base/system_parameter.h"
  - "src/communication/network_interface_cl.c"
  - "src/communication/network_interface_cl.h"
  - "src/communication/network_sr.c"
  - "src/compat/db.h"
  - "src/compat/db_obj.c"
  - "src/compat/dbi_compat.h"
  - "src/compat/dbtype_def.h"
  - "src/executables/csql_result.c"
  - "src/executables/unload_object.c"
  - "src/object/class_object.c"
  - "src/object/class_object.h"
  - "src/object/object_accessor.c"
  - "src/object/object_primitive.c"
  - "src/object/object_template.c"
  - "src/object/object_template.h"
  - "src/object/schema_class_truncator.cpp"
  - "src/object/schema_manager.c"
  - "src/object/schema_manager.h"
  - "src/object/schema_system_catalog.cpp"
  - "src/object/schema_system_catalog_constants.h"
  - "src/object/schema_system_catalog_install.cpp"
  - "src/object/schema_system_catalog_install.hpp"
  - "src/object/schema_system_catalog_install_query_spec.cpp"
  - "src/object/schema_template.c"
  - "src/object/schema_template.h"
  - "src/object/transform.c"
  - "src/object/trigger_manager.c"
  - "src/optimizer/histogram/histogram_builder.cpp (new, 265 LOC)"
  - "src/optimizer/histogram/histogram_builder.hpp (new, 63 LOC)"
  - "src/optimizer/histogram/histogram_cl.cpp (new, 1882 LOC)"
  - "src/optimizer/histogram/histogram_cl.hpp (new, 303 LOC)"
  - "src/optimizer/histogram/histogram_reader.cpp (new, 343 LOC)"
  - "src/optimizer/histogram/histogram_reader.hpp (new, 201 LOC)"
  - "src/optimizer/query_graph.c"
  - "src/optimizer/query_graph.h"
  - "src/optimizer/query_planner.c"
  - "src/optimizer/query_planner.h"
  - "src/parser/csql_grammar.y"
  - "src/parser/csql_lexer.l"
  - "src/parser/name_resolution.c"
  - "src/parser/parse_tree.h"
  - "src/parser/parse_tree_cl.c"
  - "src/parser/parser_support.c"
  - "src/parser/semantic_check.c"
  - "src/parser/xasl_generation.c"
  - "src/query/execute_schema.c"
  - "src/query/execute_schema.h"
  - "src/query/execute_statement.c"
  - "src/query/execute_statement.h"
  - "src/query/scan_manager.c"
  - "src/sp/sp_catalog.cpp"
  - "src/storage/btree.c"
  - "src/storage/heap_file.c"
  - "src/storage/oid.c"
  - "src/storage/oid.h"
  - "src/storage/statistics.h"
  - "src/transaction/boot_cl.c"
  - "src/transaction/log_applier.c"
related_components:
  - "[[components/optimizer]]"
  - "[[components/query-executor]]"
  - "[[components/scan-manager]]"
  - "[[components/heap-file]]"
  - "[[components/parser]]"
  - "[[components/parse-tree]]"
  - "[[components/semantic-check]]"
  - "[[components/name-resolution]]"
  - "[[components/execute-schema]]"
  - "[[components/execute-statement]]"
  - "[[components/schema-manager]]"
  - "[[components/system-catalog]]"
  - "[[components/object]]"
  - "[[components/system-parameter]]"
  - "[[components/btree]]"
related_sources:
  - "[[sources/cubrid-src-query]]"
  - "[[sources/cubrid-src-object]]"
  - "[[sources/cubrid-src-parser]]"
  - "[[sources/cubrid-src-storage]]"
  - "[[sources/cubrid-src-compat]]"
ingest_case: open
triggered_baseline_bump: false
baseline_before: "175442fc858bd0075165729756745be6f8928036"
baseline_after: "175442fc858bd0075165729756745be6f8928036"
reconciliation_applied: false
reconciliation_applied_at:
incidental_enhancements_count: 5
tags:
  - pr
  - cubrid
  - optimizer
  - histogram
  - statistics
  - selectivity
  - ddl
  - catalog
  - sampling
  - open
created: 2026-04-24
updated: 2026-04-24
status: open
---

# PR #6753 — Add Optimizer Histogram Support

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `OPEN` · **Author:** `@sohee-dgist` · **Jira:** [CBRD-26202](https://jira.cubrid.org/browse/CBRD-26202)
> **Base → Head:** `develop` (`1da6caa7`) → `CUBRID-HISTOGRAM` (`9970b72a`)
> **Scale:** 64 files changed, +5335 / −145 (5480 LOC total). Largest new file: `histogram_cl.cpp` (1882 LOC). 6 files over 500 LOC. Well past Scale-rule threshold — analyzed via 5 parallel deep-read subagents.
> **Approvals at snapshot:** @hornetmj, @beyondykk9, @HyunukLee, @shparkcubrid, @Hamkua, @youngjinj.

> [!note] Ingest classification: `open`
> No component pages were edited for PR-induced changes. A full Reconciliation Plan is written below (executable when the PR lands and the baseline is bumped). Incidental wiki enhancements from baseline analysis WERE applied — see the section near the bottom.

## Summary

Introduces a full histogram-based selectivity framework into the client-side query optimizer. Adds a new `_db_histogram` system catalog class, a client-executed build pipeline that runs SQL (CTE) to bucket column values with MCV (most-common-value) classification, a binary blob format (magic `"HST1"`, v1, big-endian-on-wire) stored as `VARBIT(1073741823)`, and a stateful reader that plugs histograms into `qo_expr_selectivity` / `qo_comp_selectivity` / `qo_range_selectivity` / `qo_like_selectivity`. Grammar gains three DDL verbs — `ANALYZE TABLE … UPDATE HISTOGRAM [ON cols] [WITH n BUCKETS] [WITH FULLSCAN]`, `ANALYZE TABLE … DROP HISTOGRAM [ON cols]`, `SHOW HISTOGRAM tbl [ON cols]`. Sampling scan gets a Poisson-distributed page-skip and a new weight formula (33% target, clamped 100–10000 pages). One new system parameter `default_histogram_bucket_count` (default 300, range 300–1000). All histogram construction and consumption lives in CS/SA builds; the server only stores the blob and answers catalog lookups.

## Motivation

Selectivity estimation for equality and range predicates in the baseline CUBRID planner is built on per-column scalar statistics (cardinality, unique counts, min/max). Distribution shape is invisible — skewed columns (category enums, power-law keys, geospatial clusters) get mis-estimated and produce wrong-index or wrong-join-order plans. Histograms model the distribution with a bounded number of equi-depth buckets augmented by MCV buckets (ndv=1, top-N by frequency), letting the planner reason about fraction-of-rows below/above a specific value. The PR targets Korean benchmark workloads (JOB — Join Order Benchmark — mentioned in review discussion) where baseline CUBRID picks pessimal plans that OLAP-class workloads reveal quickly.

The implementation choice to build and consume histograms **client-side** follows CUBRID's existing optimizer architecture: the parser and planner are `#if !defined(SERVER_MODE)`-gated, and this PR keeps that invariant. The server stores the blob as a normal VARBIT cell in `_db_histogram` and is otherwise unaware.

## Changes

### Structural

**New files** (all under `src/optimizer/histogram/`; the location moved from `src/histogram/` to `src/optimizer/histogram/` during review per @shparkcubrid's request):

| File | LOC | Role |
|---|---|---|
| `histogram_cl.cpp` | 1882 | analyze orchestrator, selectivity routines, dump, query-template interpolation, key extraction, per-type frac math |
| `histogram_cl.hpp` | 303 | public API declarations, query templates, `hist::histogram_key` + `histogram_key_kind` enum |
| `histogram_builder.cpp` | 265 | `hist::HistogramBuilder` — serialization-only; SQL does the actual bucketing |
| `histogram_builder.hpp` | 63 | `hist::Bucket`, `HistogramTypes` variant |
| `histogram_reader.cpp` | 343 | `hist::HistogramReader` + `HeaderV1` decoding, binary search, `find_bucket_and_check` |
| `histogram_reader.hpp` | 201 | blob layout doc (header, bucket records, string blob) |

**New system catalog class** `_db_histogram` (installed via `system_catalog_initializer::get_histogram()` at `schema_system_catalog_install.cpp:1193`):

| Column | Type | Role |
|---|---|---|
| `class_of` | `OBJECT` (general domain) | owning class MOP |
| `key_attr` | `VARCHAR(255)` | attribute name |
| `with_fullscan` | `INTEGER` | 0=sampling, 1=full |
| `null_frequency` | `DOUBLE` | null fraction |
| `histogram_values` | `BIT VARYING(1073741823)` | serialized blob (≈128 MB max) |

Constraint: `UNIQUE(class_of, key_attr)`. No primary key. No foreign key to `_db_class` (cascades are manual — see Behavioral).

**New view** `db_histogram` — projects metadata (`class_name`, `key_attr`, `with_fullscan` stringified to `'sampling scan' | 'full scan'`, `null_frequency`). Deliberately omits `histogram_values`. Forced `ORDER BY`.

**New types / macros**:
- `HIST_STATS` in `storage/statistics.h`: `{int n_attrs; DB_VALUE **histogram; double *null_frequency;}` — parallel to `ATTR_STATS`.
- `SM_CLASS::histogram` field (`class_object.h:785`) — client-side cache.
- `PT_NAME_INFO::histogram` (`DB_VALUE *`) and `PT_NAME_INFO::null_frequency` (`double`) — propagated through `qo_get_attr_info` to let every `PT_NAME` carry its column's blob and null-fraction.
- `QO_GET_HIST_STATS(entryp)` macro (`query_graph.h`) — no `self_allocated` branch, unlike `QO_GET_CLASS_STATS`.
- `MAX_HEAP_SAMPLING_PAGES = 10000`, `MIN_HEAP_SAMPLING_PAGES = 100` (`statistics.h:39-40`) — replace the single `NUMBER_OF_SAMPLING_PAGES = 5000` constant.
- `stats_free_histogram_and_init[_and_set_null]` macros (`statistics.h:60-66,117-124`).

**New parse-tree nodes** (`parse_tree.h`): `PT_UPDATE_HISTOGRAM`, `PT_DROP_HISTOGRAM`, `PT_SHOW_HISTOGRAM`. New struct `pt_histogram_info { target_table_spec, target_columns, bucket_count, with_fullscan }`.

**New statement-type enum values** (`compat/dbtype_def.h`, ABI-appended correctly): `CUBRID_STMT_UPDATE_HISTOGRAM`, `CUBRID_STMT_SHOW_HISTOGRAM`, `CUBRID_STMT_DROP_HISTOGRAM`. Alias `SQLX_CMD_DROP_HISTOGRAM` added in `dbi_compat.h` — UPDATE and SHOW aliases **missing** (inconsistency).

**New system parameter** `default_histogram_bucket_count` (`system_parameter.c:5316-5330`): default 300, min 300, max 1000, `PRM_FOR_CLIENT | PRM_USER_CHANGE`. Client-only. Default is not 30 as an early review thread suggested — it's 300.

**New reserved words** (`csql_grammar.y` + `csql_lexer.l`): `HISTOGRAM` and `BUCKETS` — both added as fully-reserved tokens with no `identifier:` fallback rule. **Breaking change** — see Behavioral.

**Signature changes (ABI-relevant)**:
- `dbt_create_object_internal(DB_OBJECT *classobj)` → `dbt_create_object_internal(DB_OBJECT *classobj, bool is_read_only)` — every caller tree-wide updated (`trigger_manager.c`, `sp_catalog.cpp`, `execute_statement.c`, `log_applier.c`, `obj_create`, `obj_copy`). Only caller that currently passes `true` is the new `obj_find_multi_attr` in `object_accessor.c`.
- `make_template(MOP, MOP, bool is_read_only)` in `object_template.c`.
- `obt_def_object(MOP, bool is_read_only)` in `object_template.c`.
- `do_create_midxkey_for_constraint` promoted from `static` to extern (`execute_statement.c/h`).
- `qo_classify` + `PRED_CLASS` enum promoted from `static`/file-local to `extern` / public `query_planner.h`.
- 9 `DEFAULT_*_SELECTIVITY` `#define`s moved from `query_planner.c` to `query_planner.h`.

### Per-file notes

- `src/optimizer/histogram/*` — entire new subsystem ([[components/optimizer]] — new sub-page [[components/optimizer-histogram]] to be created on merge)
- `src/optimizer/query_planner.c` (+414/−50) — selectivity wiring: `qo_equal_selectivity`, `qo_comp_selectivity`, `qo_range_selectivity` rewritten; new `qo_like_selectivity` ([[components/optimizer]])
- `src/optimizer/query_planner.h` (+23) — public-header promotion of `DEFAULT_*_SELECTIVITY`, `PRED_CLASS`, `qo_classify` ([[components/optimizer]])
- `src/optimizer/query_graph.c` (+22/−1) — `set_seg_node` copies histogram+null_frequency onto every PT_NAME occurrence; `qo_get_attr_info` fetches from `SM_CLASS::histogram` ([[components/optimizer]])
- `src/optimizer/query_graph.h` (+2) — `QO_GET_HIST_STATS` macro ([[components/optimizer]])
- `src/parser/csql_grammar.y` (+93) — three new productions; HISTOGRAM/BUCKETS as reserved tokens ([[components/parser]])
- `src/parser/csql_lexer.l` (+2) — case-insensitive lexer rules for the two tokens ([[components/parser]])
- `src/parser/parse_tree.h` (+16) — enum values, `PT_HISTOGRAM_INFO` union, two new `PT_NAME_INFO` fields ([[components/parse-tree]])
- `src/parser/parse_tree_cl.c` (+125) — `pt_print_analyze_histogram`, `pt_print_show_histogram`, `pt_apply_update_histogram` ([[components/parse-tree]])
- `src/parser/semantic_check.c` (+122) — `pt_check_update_histogram` (shared for all three nodes); dispatch in `pt_check_with_info` ([[components/semantic-check]])
- `src/parser/name_resolution.c` (+18/−1) — `pt_bind_names` case; `pt_make_subclass_list` zeros `histogram` ([[components/name-resolution]])
- `src/parser/xasl_generation.c` (+6) — NULL-guard on `pt_make_class_access_spec` return — collateral robustness, not histogram dispatch ([[components/xasl-generation]])
- `src/query/execute_schema.c` (+512/−1) — `do_update_histogram`, `do_drop_histogram`, `do_show_histogram`, `update_or_drop_histogram_helper`; ALTER/DROP cascade ([[components/execute-schema]])
- `src/query/execute_schema.h` (+9) — `DO_HISTOGRAM { CREATE, DROP, SHOW }` enum + prototypes ([[components/execute-schema]])
- `src/query/execute_statement.c` (+96/−28) — PT_*_HISTOGRAM dispatch across 5 switches; `do_create_midxkey_for_constraint` extern ([[components/execute-statement]])
- `src/query/execute_statement.h` (+5) — new externs ([[components/execute-statement]])
- `src/query/scan_manager.c` (+8/−2) — sampling weight formula: `MAX(MIN(3, ceil(total/100)), MAX(total/10000, 1))` ([[components/scan-manager]])
- `src/storage/heap_file.c` (+22/−1) — `random_poisson_weight(int weight)` with `thread_local std::mt19937{123456789u}`; call-site in `heap_next_internal` ([[components/heap-file]])
- `src/storage/btree.c` (+2/−1) — `OID_CACHE_HISTOGRAM_CLASS_ID` added to snapshot-skip predicate in `xbtree_find_unique` ([[components/btree]])
- `src/storage/statistics.h` (+19/−2) — `HIST_STATS`, `MAX_HEAP_SAMPLING_PAGES`, `MIN_HEAP_SAMPLING_PAGES`; `stats_adjust_sampling_weight` uses new max ([[components/storage]])
- `src/storage/oid.c` (+3), `src/storage/oid.h` (+1) — new OID cache slot `OID_CACHE_HISTOGRAM_CLASS_ID` (**never populated at runtime** — see supplementary findings) ([[components/storage]])
- `src/object/schema_system_catalog_install.cpp` (+75) — `_db_histogram` table + `db_histogram` view definitions; registration in `catcls_init` ([[components/system-catalog]])
- `src/object/schema_system_catalog_install.hpp` (+3) — two new static decls ([[components/system-catalog]])
- `src/object/schema_system_catalog_install_query_spec.cpp` (+24) — view projection SQL ([[components/system-catalog]])
- `src/object/schema_system_catalog_constants.h` (+35) — `CT_HISTOGRAM_NAME`, `CTV_HISTOGRAM_NAME`, `CNT_CATCLS_OBJECTS = 8` (was 6) ([[components/system-catalog]])
- `src/object/schema_system_catalog.cpp` (+4/−2) — histogram catalog name registration in two lookup tables ([[components/system-catalog]])
- `src/object/schema_manager.c` (+202/−1) — `sm_add_histogram`, `sm_drop_histogram`; cascade in `sm_delete_class_mop`; invalidation in `install_new_representation`, `sm_update_statistics` ([[components/schema-manager]])
- `src/object/schema_manager.h` (+2) — two new prototypes ([[components/schema-manager]])
- `src/object/schema_template.c` (+170) — `smt_add_histogram`, `smt_check_histogram_exist`, `smt_check_histogram_exist_and_delete` ([[components/schema-manager]])
- `src/object/schema_template.h` (+4) — three new prototypes ([[components/schema-manager]])
- `src/object/class_object.c` (+7) — `SM_CLASS::histogram` init/free ([[components/object]])
- `src/object/class_object.h` (+3) — new field + `classobj_check_histogram_exist` prototype (**declared but undefined** at head) ([[components/object]])
- `src/object/object_template.c` (+12/−6) — `is_read_only` plumbing in `make_template` and `obt_def_object` ([[components/object]])
- `src/object/object_template.h` (+1/−1) — `obt_def_object` signature ([[components/object]])
- `src/object/object_accessor.c` (+56/−17) — `obj_find_multi_attr` rewrite: builds read-only template → `do_create_midxkey_for_constraint` → `btree_find_unique` ([[components/object]])
- `src/object/object_primitive.c` (+30/−2) — `mr_index_writeval_object` split from `mr_index_writeval_oid` for `DB_TYPE_OBJECT` client-side path ([[components/db-value]])
- `src/object/schema_class_truncator.cpp` (+1/−16) — `CNT_CATCLS_OBJECTS` relocated to header; comment block removed ([[components/schema-manager]])
- `src/object/transform.c` (+1) — whitespace touch (author agreed to remove in refactor) ([[components/object]])
- `src/object/trigger_manager.c` (+1/−1) — mechanical `dbt_create_object_internal(..., false)` update ([[components/object]])
- `src/sp/sp_catalog.cpp` (+3/−3) — three mechanical updates ([[components/sp]])
- `src/compat/db.h` (+1/−1) — `dbt_create_object_internal` signature change — **ABI break for out-of-tree consumers** ([[components/dbi-compat]])
- `src/compat/db_obj.c` (+5/−3) — `db_find_multi_unique` adds unconditional `er_clear` (masks real errors); `dbt_create_object` defaults to `false` ([[components/compat]])
- `src/compat/dbtype_def.h` (+4/−1) — three new `CUBRID_STMT_*` enum members appended ([[components/dbi-compat]])
- `src/compat/dbi_compat.h` (+1) — `SQLX_CMD_DROP_HISTOGRAM` alias only (UPDATE/SHOW missing) ([[components/dbi-compat]])
- `src/communication/network_interface_cl.c` (+47) — `update_histogram_for_all_classes()` — client-only (misleadingly placed in network_interface_cl) ([[components/request-response]])
- `src/communication/network_interface_cl.h` (+1) — prototype ([[components/request-response]])
- `src/communication/network_sr.c` (+1) — unrelated fix: `NET_SERVER_BTREE_FIND_MULTI_UNIQUES` `action_attribute = IN_TRANSACTION` (pre-existing latent bug, surfaced by histogram catalog) ([[components/communication]])
- `src/base/system_parameter.c` (+15/−1), `system_parameter.h` (+2) — `default_histogram_bucket_count` ([[components/system-parameter]])
- `src/base/ddl_log.c` (+2) — `PT_UPDATE_HISTOGRAM` + `PT_DROP_HISTOGRAM` recognized as DDL; SHOW correctly excluded ([[components/base]])
- `src/transaction/boot_cl.c` (+3) — two catalog name entries in destroy-prohibited lists; one blank line (`ENABLE_UNUSED_FUNCTION` section — author noted as "임시 통일성") ([[components/server-boot]])
- `src/transaction/log_applier.c` (+5/−2) — two mechanical `dbt_create_object_internal` updates; `la_apply_statement_log` switch adds UPDATE+DROP (SHOW correctly excluded) ([[components/log-manager]])
- `src/executables/csql_result.c` (+2) — CSQL display strings for UPDATE+DROP (SHOW missing — inconsistency) ([[components/csql-shell]])
- `src/executables/unload_object.c` (+2) — `CT_HISTOGRAM_NAME` added to `prohibited_classes[]`; `CTV_HISTOGRAM_NAME` commented-out TODO — **unload regression**: backup/restore loses histograms ([[components/utility-binaries]])
- `CMakeLists.txt` (+2) — histogram include dir ([[modules/src]])
- `cs/CMakeLists.txt` (+15), `sa/CMakeLists.txt` (+14) — histogram sources linked into `cubridcs` and `cubridsa` only; **no entries in any server binary**

### Behavioral

1. **Grammar — new DDL (reserved-word impact)**:
   - `ANALYZE TABLE tbl UPDATE HISTOGRAM [ON col[, col]*] [WITH n BUCKETS] [WITH FULLSCAN]` — clause order is fixed; `WITH 0 BUCKETS` silently falls back to default 300.
   - `ANALYZE TABLE tbl DROP HISTOGRAM [ON col[, col]*]`.
   - `SHOW HISTOGRAM tbl [ON col[, col]*]`.
   - `HISTOGRAM` and `BUCKETS` are **fully-reserved words** — no `identifier:` fallback. **Any existing schema with a column or table named `histogram` or `buckets` fails to parse post-merge.** Not flagged by reviewers. Suggested fix: follow the `HEAP` / `FULLSCAN` template (contextual reserved via `identifier:` fallback).
2. **Selectivity integration in `query_planner.c`** — Reserved defaults stay intact as the fallback, but the following predicates now consult histograms when `PT_NAME_INFO::histogram != NULL`:
   - `PT_EQ`, `PT_NULLSAFE_EQ`, `PT_NE` (via `qo_not_selectivity`): `(bucket_rows / total_rows) / approx_ndv`, with `1/total_rows` for out-of-domain keys (divide-by-zero not guarded when `total_rows == 0`).
   - `PT_GE`, `PT_GT`, `PT_LE`, `PT_LT`: domain fraction interpolation + MCV snap.
   - `PT_RANGE` with every `PT_BETWEEN_*` variant (`GE_LE`, `GE_LT`, `GT_LE`, `GT_LT`, `INF_LE`, `INF_LT`, `GE_INF`, `GT_INF`, `EQ_NA`).
   - `PT_LIKE`: new hybrid model — MCV fast-path + per-bucket `like_match_string` against `bucket_hi` + 3-factor confidence blend.
   - `PT_IS_NULL` / `PT_IS_NOT_NULL`: use `info.name.null_frequency` directly.
   - **Not touched**: `qo_between_selectivity` still returns `DEFAULT_BETWEEN_SELECTIVITY = 0.01` (bare `PT_BETWEEN` survives into the planner unchanged); `qo_all_some_in_selectivity` (PT_IN) keeps `DEFAULT_IN_SELECTIVITY = 0.01`; `PT_LIKE_ESCAPE` keeps `PRM_ID_LIKE_TERM_SELECTIVITY`. So `x BETWEEN 1 AND 10` vs `x >= 1 AND x <= 10` vs `x LIKE 'a%' ESCAPE '\'` now behave wildly differently.
3. **`PT_IS_NULL`/`IS_NOT_NULL` post-hoc multiplier**: `qo_expr_selectivity` multiplies the operator selectivity by `(1 − null_frequency)` for every non-NULL predicate (not just `PT_EQ`), regardless of whether the histogram path succeeded. Double-counts when builder-total already excludes NULLs; adds 0.7× factor to `DEFAULT_COMP_SELECTIVITY` fallbacks on high-NULL columns.
4. **Late binding dependency**: histogram path dereferences `env->parser->host_variables[rhs->info.host_var.index]` whenever `pc_rhs == PC_HOST_VAR`. With `hostvar_late_binding = off` (default), auto-parameterized host vars have placeholder or uninitialized `DB_VALUE`s — the `histogram_extract_key` switch either fails through (`success=false` → fallback) or crashes on corrupt `general_info.type`. The PR **does not check `PRM_ID_HOSTVAR_LATE_BINDING`** and does not bounds-check `host_variables[index]`. The user comment thread makes clear the author expects `hostvar_late_binding=true` for histogram-based selectivity.
5. **Sampling scan — new weight math** (`scan_manager.c:2906`):
   ```
   base_weight = 3                                        # ~33% target
   min_weight  = ceil(total_pages / 100)                  # MIN_HEAP_SAMPLING_PAGES
   max_weight  = total_pages / 10000                      # MAX_HEAP_SAMPLING_PAGES
   weight      = max(min(base, min_w), max(max_w, 1))
   ```
   Pages actually sampled: clamped between 100 and 10000. Source code comment says "30% / max 5000 pages" — both wrong; code does 33% and max 10000.
6. **Sampling scan — Poisson skip**: `heap_next_internal` now calls `random_poisson_weight(weight)` to decide per-page stride. RNG is `thread_local std::mt19937{123456789u}` with a **fixed seed** — fully deterministic per (thread, install). Greptile flagged as a bias risk. Combined with thread-local-per-connection pooling, sample sequences repeat across restarts.
7. **`UPDATE STATISTICS` now rebuilds histograms unconditionally** via `update_histogram_for_all_classes`. Iterates every top-level class, calls `update_or_drop_histogram_helper(..., DO_HISTOGRAM_CREATE)` per class (bucket_count=−1 → clamps to 300). On a schema with N classes × M columns, cost is O(N × M) sampling scans. No sysprm gate.
8. **ALTER/DROP cascade** — histograms are dropped manually (no FK cascade because `class_of` has no FK):
   - `DROP TABLE` — `sm_delete_class_mop` iterates `class_->attributes` and `db_drop`s each histogram. **Dead null-check** can jump past `AU_ENABLE` (greptile P2 unfixed at head).
   - `ALTER TABLE … DROP COLUMN` / `MODIFY COLUMN` / `CHANGE COLUMN` / `RENAME ATTR` — `do_alter` pre-pass.
   - `ALTER TABLE … RENAME` — comment says "no effect" (correct — `class_of` is OBJECT, not name).
   - **TRUNCATE TABLE does NOT invalidate histograms.** After `TRUNCATE`, bucket counts still reference deleted rows — stale histogram misleads the planner.
9. **Privilege gap**: all three histogram DDL wrappers (`do_update_histogram`, `do_drop_histogram`, `do_show_histogram`) run their full body under `AU_DISABLE`. Neither the executor nor the semantic-checker verifies owner/grant-ALTER. **Any user with `SELECT` on a table can run `ANALYZE TABLE t DROP HISTOGRAM` on it.** Not flagged by any reviewer.
10. **Unload regression**: `cubrid unloaddb` marks `_db_histogram` as prohibited — backup/restore silently loses every histogram.
11. **Replication**:
    - `do_replicate_statement` (client) emits `PT_UPDATE_HISTOGRAM`, `PT_DROP_HISTOGRAM`, **and `PT_SHOW_HISTOGRAM`** into the replication stream. SHOW is read-only — emitting it is semantically wrong. `la_apply_statement_log` on the slave correctly omits SHOW from its switch, so replay silently ignores it, but bandwidth is wasted.
    - UPDATE/DROP replay the SQL text; each replica builds its own histogram from its own heap. Master/slave histograms diverge — probably desired but undocumented.
12. **ABI impact**:
    - `dbt_create_object_internal` signature changes — breaks any out-of-tree consumer of `libcubridcs.so` / `libcubridsa.so`. In-tree JDBC / PHP / Python drivers rebuild fine.
    - `CUBRID_STMT_*` enum: new values appended **before `CUBRID_MAX_STMT_TYPE`** — correct for ABI stability.
13. **Blob format** — magic `"HST1"`, version 1, `HeaderV1` 24 bytes. All multi-byte fields serialized via `OR_PUT_INT` / `OR_PUT_INT64` (**big-endian on wire**) despite header comment claiming "LE". Bucket records are 24-byte fixed (`data_hi`, `cumulative int64`, `approx_ndv int64`). MCV sentinel = `approx_ndv == 1`. Strings ≤4 chars stored inline in the `data_hi` slot; longer strings offset into trailing string blob. Header comment lists wrong field names (says `f64 cumulative` etc. — actual is `int64_t`).
14. **Key-kind mapping** (`is_histogrammable_type` + `histogram_extract_key`):
    - i64: `INTEGER`, `SHORT`, `BIGINT`
    - dbl: `FLOAT`, `DOUBLE`, `NUMERIC` (lossy via `numeric_coerce_num_to_double`)
    - str: `CHAR`, `VARCHAR`, `BIT`, `VARBIT` (CHAR trailing spaces NOT stripped)
    - u64: `TIME`, `TIMESTAMP`, `TIMESTAMPLTZ`, `TIMESTAMPTZ` (TZ offset discarded), `DATE`, `DATETIME` (packed `date<<32 | time`), `DATETIMETZ`, `DATETIMELTZ` (TZ discarded)
    - **Unsupported**: `MONETARY`, `ENUM`, `JSON`, `CLOB`, `BLOB`, `OBJECT`, `OID`, set types.
    - **`DATETIMETZ`/`DATETIMELTZ` latent bug**: `is_histogrammable_type` accepts them and `histogram_extract_key` has cases, but `histogram_builder.cpp` switch `default`-falls and returns NULL. `analyze_classes` writes null_frequency first, then fails on the blob build — catalog row ends up partial (null blob, non-null null_frequency).
15. **MCV classification rule**: MCV threshold is `0.5 / bucket_count` — a value is MCV-eligible if its frequency exceeds half-of-equal-share. For `bucket_count=3` → 16.5%. With default `bucket_count=300` → `0.00167` (0.17%). Top-N-by-count selection means non-distinct values can land as MCV if the top-N list is shorter than the qualifying set — acknowledged pathology.

### New surface (no existing wiki reference)

- `src/optimizer/histogram/` entire directory — **no existing wiki page**. Need to create `components/optimizer-histogram` on merge.
- `_db_histogram` system catalog class — not mentioned in `components/system-catalog.md`.
- `HIST_STATS` struct — not in `sources/cubrid-src-storage.md`.
- `SM_CLASS::histogram` field — not in `components/object.md` or `components/schema-manager.md`.
- `OID_CACHE_HISTOGRAM_CLASS_ID` enum value — not in any `storage/oid` reference.
- `random_poisson_weight` helper in `heap_file.c` — not in `components/heap-file.md`.
- `default_histogram_bucket_count` sysprm — not in `components/system-parameter.md`.
- New `PT_NAME_INFO::{histogram, null_frequency}` fields — not in `components/parse-tree.md`.
- Three new `PT_*_HISTOGRAM` parse-tree node types — not in `components/parse-tree.md`.

## Review discussion highlights

Only authoritative / design-rationale signals. Bot nits, CI noise, and "/run all" skipped.

- **Module location** (`@shparkcubrid`, `CMakeLists.txt:416`): move from `src/histogram/` → `src/optimizer/histogram/`. @sohee-dgist agreed; confirmed at head.
- **Default bucket count** (`@shparkcubrid`, `csql_grammar.y:4751`): MySQL uses 100. @sohee-dgist: default is 30 (at the time). @shparkcubrid later: "왜 30개? 다른 DBMS에 비하면 작아 보이네요." @sohee-dgist: "30개도 많다고 명환님이랑 박사님이 말씀하셨던 기억이 … 254 or 100 따를까요?" Resolution: ended at 300 via `default_histogram_bucket_count` sysprm. Review history is stale — no reviewer has seen the final 300.
- **`MAX/MIN_HEAP_SAMPLING_PAGES` as defines** (`@shparkcubrid`, `scan_manager.c:2905`): @sohee-dgist confirmed fixed in commit `c0ae69a2`. Final values: `MAX=10000`, `MIN=100`.
- **`number_of_mcv` spec in catalog** (`@shparkcubrid`, `schema_system_catalog_install.cpp:1274`): user-visible format for `histogram_values` required. Resolution: added `SHOW HISTOGRAM` grammar. Jira Implementation updated.
- **`null_frequency` display** (`@shparkcubrid`, `schema_system_catalog_install_query_spec.cpp:1653`): change scientific to plain decimal (`2e-02` → `0.02`). @sohee-dgist: will apply. Not yet applied at head; current view casts to `NUMERIC(18,12)` which helps but may still yield scientific formatting on display.
- **`make_template_for_read_only` pattern** (`@shparkcubrid`, `object_template.c:1606`): use a flag parameter instead of a separate function. @sohee-dgist: will apply. At head this IS the shape — `make_template(MOP, MOP, bool is_read_only)` and the `dbt_create_object_internal` signature change follows that. The still-open greptile bug about `au_fetch_instance` still calling `AU_FETCH_UPDATE` inside the read-only path is latent (only caller passes `object=NULL` so fetch branch is bypassed).
- **`cs/CMakeLists.txt` builder placement** (`@xmilex-git`, line 432): "cs 모듈에 builder는 왜 필요한건가요?" @sohee-dgist: "빌드와 리드가 모두 클라이언트에서 이루어집니다." Confirms client-only architecture.
- **Reserved word `role` collision** (`@xmilex-git`, Feb-12): `analyze table role_type update histogram on role;` failed to parse. @sohee-dgist fixed in commit `16c0c34`. Hints at the broader reserved-word sensitivity which the PR never fully resolved for `HISTOGRAM`/`BUCKETS`.
- **Null-all column `null_frequency`** (`@xmilex-git`, Feb-23): observed 25% instead of 100% for all-null column. @sohee-dgist narrowed to sampling behavior; fixed through commit `b0c3e27e`-era.
- **Jira testing gate** (`@xmilex-git`, PR body): `SET SYSTEM PARAMETERS 'hostvar_late_binding=true';` required for histogram-based selectivity. This is the hostvar-late-binding dependency explicitly mentioned in behavioral note 4 above.
- **Full `SHOW HISTOGRAM` format** (`@shparkcubrid`, later): user wanted a one-look dump with all bucket endpoints, rows, ndv, cumulative. @sohee-dgist delivered the bordered ASCII-frame dump in `dump_histogram`.
- **MCV correctness** (`@shparkcubrid`, `histogram_cl.hpp:66`): two specific pathologies — (a) unique values can be tagged MCV because "top-N" rule doesn't filter by threshold; (b) display shows endpoint of neighbor MCV as own endpoint. @sohee-dgist acknowledged — display quirk (b) fixed; (a) "추후 개선 PR을 통해 고도화" deferred to a follow-up.

Approvals: 6 (@hornetmj 03-10, @beyondykk9 03-18, @HyunukLee 03-24, @shparkcubrid 04-01, @Hamkua 04-03, @youngjinj 04-15). Not yet merged at ingest time.

## Reconciliation Plan

Executable post-merge (or on explicit request `apply reconciliation for PR #6753`). Organized page-by-page. Each entry gives the concrete before/after excerpt so a future reconciliation pass does not need to re-read the PR.

### [[components/optimizer]] — Selectivity model + histogram wiring

> Current state: page is a 47-line stub with no content about selectivity estimation.

- **Current claim:** "Selects join orders, access paths, and physical operators based on cost estimates."
- **Proposed replacement:** add a "Selectivity model" section after "Side of the wire". Content:
  - `PT_NAME_INFO::histogram` (DB_VALUE *) + `PT_NAME_INFO::null_frequency` (double) carry per-column blob pointer + null-fraction on every PT_NAME reference.
  - `qo_expr_selectivity` dispatches per PT-operator:
    - Equality/inequality → `histogram_get_equal_selectivity` (fallback `DEFAULT_EQUAL_SELECTIVITY=0.001`)
    - Comparison → `histogram_get_comp_selectivity(is_ge, include_equal)` (fallback `DEFAULT_COMP_SELECTIVITY=0.1`)
    - Range (PT_RANGE) → per-BETWEEN-case pair of comp calls (fallback `DEFAULT_BETWEEN_SELECTIVITY=0.01`)
    - LIKE → `histogram_get_like_selectivity` (fallback `PRM_ID_LIKE_TERM_SELECTIVITY`)
    - IS [NOT] NULL → `null_frequency` directly (fallback `DEFAULT_NULL_SELECTIVITY=0.01`)
    - IN, plain BETWEEN, LIKE ESCAPE → **not yet histogram-aware**, keep defaults.
  - Post-hoc multiplier: `selectivity *= (1 − null_frequency)` applied to all non-NULL predicates regardless of histogram success.
  - `DEFAULT_*_SELECTIVITY` and `PRED_CLASS` are public in `query_planner.h` since this PR.
- **Callout:** `[!update]` with "PR #6753 / merge `<short-sha>` — selectivity redesigned; see [[prs/PR-6753-optimizer-histogram-support]]."

### [[components/optimizer-histogram]] — NEW sub-page (to be created on merge)

One new component page. Placeholder skeleton the reconciliation pass should materialize:
- path: `src/optimizer/histogram/`
- six files (listed in frontmatter)
- public surface (listed in Structural)
- blob layout (magic HST1, 24-byte HeaderV1, 24-byte bucket records, trailing string blob)
- key-kind mapping table (Behavioral §14)
- MCV classification rule (Behavioral §15)
- client-only architecture note
- SQL-driven build pipeline (`MCV_COUNT_QUERY_TEMPLATE`, `HISTOGRAM_QUERY_TEMPLATE`, `NULL_FREQUENCY_*_QUERY_TEMPLATE`)
- known limitations: `DATETIMETZ`/`DATETIMELTZ` partial-write; no TRUNCATE invalidation; unload regression; no privilege check.

### [[components/parser]] — Three new statements + two new reserved words

- **Current state:** lists existing statement keyword surface.
- **Add:** `ANALYZE TABLE … UPDATE|DROP HISTOGRAM …` and `SHOW HISTOGRAM …` grammar productions. Note clause-order requirement. Note `HISTOGRAM` and `BUCKETS` as **fully-reserved** words, with the explicit caveat that this breaks backward-compatibility for schemas using either name.
- **Callout:** `[!update]` with merge-sha; add separate `[!warning] Breaking change` callout for the reserved-word impact.

### [[components/parse-tree]] — New nodes + new `PT_NAME_INFO` fields

- **Add:** three new `PT_NODE_TYPE` values (UPDATE/SHOW/DROP HISTOGRAM), `PT_HISTOGRAM_INFO` union member, two new `PT_NAME_INFO` fields (`histogram` DB_VALUE*, `null_frequency` double).
- **Note:** `pt_print_analyze_histogram` / `pt_print_show_histogram` pretty-printers drop `WITH n BUCKETS` and `WITH FULLSCAN` clauses — round-trip broken. Flag as known limitation.

### [[components/semantic-check]] — `pt_check_update_histogram`

- **Add:** new semantic check for all three histogram node types. Validates target is non-synonym, non-virtual, non-partition class. Does NOT validate column existence at semantic time — errors surface later in executor via generic name-resolution or `is_histogrammable_type`.

### [[components/name-resolution]] — Scope-bind for histogram DDL

- **Add:** `pt_bind_names` dispatches histogram nodes by pushing `target_table_spec` on the scope stack and walking `target_columns` as attribute leaves. `pt_make_subclass_list` zeros the new `PT_NAME_INFO::histogram` pointer (defensive NULL-init; `null_frequency` left to `parser_new_node` calloc — fragile).

### [[components/execute-schema]] — New `DO_HISTOGRAM` orchestrator + ALTER cascade

- **Add:** `DO_HISTOGRAM { CREATE, DROP, SHOW }` enum. Four new functions: `do_update_histogram`, `do_drop_histogram`, `do_show_histogram`, `update_or_drop_histogram_helper`. ALTER cascade logic in `do_alter` preface. Privilege gap note: `AU_DISABLE` wraps the full body with no explicit ownership/grant check.
- **Callout:** `[!warning]` for the privilege gap.

### [[components/execute-statement]] — Five-switch dispatch

- **Add:** PT_*_HISTOGRAM cases in `do_statement` body, `do_execute_statement` fetch-version switch + savepoint switch + body, `do_replicate_statement` (with note that `PT_SHOW_HISTOGRAM` inclusion is a bug — SHOW is read-only). `do_create_midxkey_for_constraint` now extern.

### [[components/schema-manager]] — `sm_{add,drop}_histogram` + cache

- **Add:** `sm_add_histogram`, `sm_drop_histogram`, `smt_add_histogram`, `smt_check_histogram_exist[_and_delete]` — full signatures, savepoint names (`"aDDhISTOGRAM"`, `"dELETEhISTOGRAM"`), double-lookup anti-pattern in `sm_drop_histogram`, silent no-op when histogram doesn't exist (inconsistent with `sm_drop_constraint`).
- **Cache:** `SM_CLASS::histogram` (client-side only, lazy-loaded in `sm_get_class_with_statistics`, invalidated in `install_new_representation` / `sm_update_statistics` / end of `update_or_drop_histogram_helper`). Coarse-grained — any stats refresh blows away the full histogram cache.

### [[components/system-catalog]] — `_db_histogram` and `db_histogram`

- **Add:** `_db_histogram` table (columns + UNIQUE(class_of, key_attr), no PK, no FK). `db_histogram` view (projection, no blob exposed). `CT_HISTOGRAM_NAME` / `CTV_HISTOGRAM_NAME` constants. `CNT_CATCLS_OBJECTS = 8` (was 6). Note the no-PK / no-FK choice means cascades are manual and orphan rows are possible on crash recovery mid-DROP-TABLE.

### [[components/object]] — `SM_CLASS::histogram` + `is_read_only` template

- **Add:** new `SM_CLASS::histogram` cache field. `make_template` / `obt_def_object` / `dbt_create_object_internal` now carry `bool is_read_only` — only caller passing `true` is `obj_find_multi_attr`. Greptile-flagged latent bug: internal `au_fetch_instance` still uses `AU_FETCH_UPDATE` in the read-only path — harmless today (single caller passes NULL object), will trip any future caller that passes a non-NULL object.
- **`obj_find_multi_attr` rewrite**: uses template + `do_create_midxkey_for_constraint` + `btree_find_unique` instead of `obj_find_object_by_cons_and_key`; does NOT lock/fault the returned MOP (caller must `au_fetch_instance` on demand).
- **`mr_index_writeval_object` split**: separated from `mr_index_writeval_oid` to handle `DB_TYPE_OBJECT` client-side via `WS_OID`. Server-side code path leaves `oidp = NULL` — defensive assert absent.

### [[components/dbi-compat]] — ABI-break + new statement-type enums

- **Add:** `dbt_create_object_internal` signature-change note (ABI break for out-of-tree `libcubridcs` consumers). Three new `CUBRID_STMT_*` enum values (ABI-safe — appended). `SQLX_CMD_DROP_HISTOGRAM` alias only (UPDATE / SHOW aliases missing — inconsistency).

### [[components/scan-manager]] — Sampling weight formula

- **Current:** mentions `S_HEAP_SAMPLING_SCAN` but no math.
- **Add:** after the scan-types section, document the new `scan_open_heap_scan` sampling-weight calculation (target 33%, clamp 100..10000 pages via `MIN/MAX_HEAP_SAMPLING_PAGES`). Note the in-code comment claims "30% / max 5000" — both wrong.

### [[components/heap-file]] — Poisson skip in sampling scan

- **Add:** `random_poisson_weight(int weight)` helper (`heap_file.c:7881-7898`) using `thread_local std::mt19937{123456789u}` with fixed seed. Shifted Poisson (`lambda = weight-1`, result `+1`). Known bias risk: fixed seed means every thread across every restart produces the same skip sequence.

### [[components/btree]] — MVCC snapshot skip for `_db_histogram`

- **Add:** `xbtree_find_unique` snapshot-skip predicate extended to include `OID_CACHE_HISTOGRAM_CLASS_ID`. Rationale: histogram rows written in a session's DDL must be visible to the same session's follow-up lookups regardless of snapshot policy.
- **Known bug:** the `oid_Histogram_class` OID is never populated by `boot_client_find_and_cache_class_oids` (see supplementary findings §10) — the snapshot-skip will never fire because `oid_check_cached_class_oid` compares against the zero OID. File as `[!gap]`.

### [[components/storage]] — `oid_Histogram_class` cache + `HIST_STATS`

- **Add:** `OID_CACHE_HISTOGRAM_CLASS_ID` enum value, `oid_Histogram_class` global, `oid_Cache[]` table entry. Flag: never populated at runtime (latent bug).
- **Add:** `HIST_STATS` struct definition, `MAX/MIN_HEAP_SAMPLING_PAGES` constants, new `stats_adjust_sampling_weight` threshold (was 1000, now 2000 — 2× stricter).

### [[components/system-parameter]] — `default_histogram_bucket_count`

- **Add:** new client-only parameter, default/min=300, max=1000, user-changeable. Note review thread about whether `default_` prefix is needed — unresolved.

### [[components/utility-binaries]] — unload regression

- **Add:** `_db_histogram` prohibited from unload (`unload_object.c` prohibited_classes), so `cubrid unloaddb` silently drops all histograms. Backup/restore loses histograms.
- **Callout:** `[!warning]` for the backup/restore loss.

### [[components/csql-shell]] — missing SHOW display string

- **Add:** new display strings for UPDATE_HISTOGRAM and DROP_HISTOGRAM; SHOW falls through to default naming (inconsistency).

### [[components/log-manager]] — HA replay of histogram DDL

- **Add:** `la_apply_statement_log` handles `CUBRID_STMT_UPDATE_HISTOGRAM` and `CUBRID_STMT_DROP_HISTOGRAM` via the DDL-replay path (replays SQL text, not blobs). SHOW correctly excluded. Slaves build their own histograms from their own heaps — master/slave divergence is intentional-but-undocumented.

### [[components/base]] — ddl_log additions

- **Add:** `PT_UPDATE_HISTOGRAM` and `PT_DROP_HISTOGRAM` recognized as DDL in `logddl_is_ddl_type`.

### [[flows/ddl-execution-path]] — three new DDL verbs

- **Add:** end-to-end trace for `ANALYZE TABLE t UPDATE HISTOGRAM ...`: lexer → parser → semantic → name resolution → `do_update_histogram` → `update_or_drop_histogram_helper` → `analyze_classes` (client-side SQL via `db_compile_and_execute_local`) → `HistogramBuilder::build` → `set_histogram` → catalog write.

### [[Key Decisions]] — candidate ADR

- **Add link** to a new ADR (judgment call, after merge) covering: client-side-only architecture, blob-in-catalog over separate stats file, `0.5/B` MCV threshold, `bucket_count` default=300 rationale.

### New surface — no existing wiki reference

These need NEW pages (or sections) on merge, not just edits:

1. `components/optimizer-histogram` — the entire new subsystem.
2. `components/storage-oid-cache` (or a section inside `components/storage`) — `OID_CACHE_*` enum + population flow.
3. A `decisions/NNNN-histogram-architecture.md` ADR covering the choices listed in §Key Decisions.

## Pages Reconciled

n/a — PR is OPEN, the Reconciliation Plan above is not executed during this ingest. Promote Plan content here when `apply reconciliation for PR #6753` is invoked.

## Incidental wiki enhancements

Applied during this ingest based on baseline-only facts surfaced during the deep-read. These are **baseline truths** — they stand regardless of whether PR #6753 merges.

1. [[components/scan-manager]] — added "Sampling scan" sub-section documenting the pre-PR `S_HEAP_SAMPLING_SCAN` path, `NUMBER_OF_SAMPLING_PAGES = 5000` in baseline, and `stats_adjust_sampling_weight` NDV correction, so the page-merge Reconciliation later has anchor text to update.
2. [[components/heap-file]] — added "Sampling scan integration" note pointing at `heap_next_internal` + `sampling->weight` (existing, baseline) so the histogram supplementary changes have context.
3. [[components/optimizer]] — expanded the "Inputs / outputs" section with a brief "Selectivity defaults" paragraph enumerating the existing `DEFAULT_*_SELECTIVITY` constants and their pre-PR location in `query_planner.c` (they are currently in `.c`, slated to move to `.h` in PR #6753). This baseline fact is already missing from the stub page.
4. [[components/system-catalog]] — added note about the `CNT_CATCLS_OBJECTS` invariant (currently 6 at baseline) and its ripple into `schema_class_truncator.cpp`, since this is a baseline constant not yet documented.
5. [[components/object]] — added brief note on `mr_index_writeval_oid`'s dual acceptance of `DB_TYPE_OBJECT` and `DB_TYPE_OID` (baseline behavior, pre-split). This is used in `obj_find_multi_attr`'s midxkey path and was previously undocumented.

## Deep analysis — supplementary findings

Synthesized from 5 parallel subagent reports. These are observations that go beyond the existing bot/reviewer comments in the PR and deserve highlighting; most are unreported and should factor into either (a) the reconciliation pass or (b) a follow-up correctness PR.

### Correctness — high priority

1. **`oid_Histogram_class` never populated.** `OID_CACHE_HISTOGRAM_CLASS_ID` enum exists, `oid_Cache[]` entry exists, but `boot_client_find_and_cache_class_oids` never calls `oid_set_cached_class_oid(OID_CACHE_HISTOGRAM_CLASS_ID, …)`. Consequence: `oid_check_cached_class_oid` compares against zero-OID, so the new MVCC-snapshot-skip in `btree.c:24783` for the histogram catalog NEVER fires. The intended isolation behavior for `_db_histogram` lookups is disabled. Not flagged by any reviewer. File as `[!gap]` on the reconciliation page; recommend adding an `oid_set_cached_class_oid` call to the `boot_client_find_and_cache_class_oids` block.

2. **`update_histogram_for_all_classes` double-leaks.** Allocates `LIST_MOPS *lmops = locator_get_all_class_mops(…)` but never calls `locator_free_list_mops(lmops)` on any path. Also, on `lmops == NULL`, returns `ER_FAILED` without `AU_ENABLE(save)` — session-wide authorization bypass until the next explicit `AU_ENABLE` anywhere. Called from `sm_update_all_statistics`, i.e. `UPDATE STATISTICS;` with no class list — every admin-run statistics refresh leaks and risks auth bypass on empty-class databases.

3. **`dbt_put_internal` on a "read-only" template.** `obj_find_multi_attr` creates a template with `is_read_only=true` then calls `dbt_put_internal` on it. The `is_read_only` flag only affects `au_fetch_class` inside `make_template`; `dbt_put_internal` doesn't check it. So the name is a lie for this specific usage — the template IS written to, just never `dbt_finish_object`-ed (it's `dbt_abort_object`-ed at the end). Cognitive hazard for future callers that might try to use `is_read_only=true` as a general "I promise not to mutate" guarantee.

4. **Post-hoc NULL multiplier applies even to fallback selectivities** (query_planner.c `qo_expr_selectivity` around diff 4851-4863). When `PT_EQ`'s histogram path fails and returns `DEFAULT_EQUAL_SELECTIVITY = 0.001`, the multiplier still fires: `0.001 * (1 - null_frequency)`. A NOT NULL column with `null_frequency=0` is harmless, but a column with 30% NULLs receives a 0.7× factor on a default that never had null-aware semantics. Silently bias the cost model for every fallback site.

5. **Divide-by-zero in equality-selectivity.** `histogram_get_equal_selectivity` at `histogram_cl.cpp:829` computes `1.0 / total_rows()` with no zero check (unlike the comp helper which checks). A histogram freshly created on an empty table has `total_rows = 0` → `+inf`, then `/ approx_ndv` → NaN, then NaN propagates into plan cost. Plan selection becomes non-deterministic per-NaN.

6. **`CONST op ATTR` symmetry flipped** (query_planner.c:10037). Greptile P1 filed — at head commit, every `include_equal` is inverted. `PT_GE` when `CONST >= ATTR` should mean `ATTR <= CONST` with inclusive equal, but the code passes `(false, false)` (exclusive). Every comparison with constant-on-the-left is off-by-one on the boundary.

7. **`str`/`u64` singleton-bucket selectivity still wrong for `>`**. For `i64`/`dbl` the strict-`>` case uses `is_ge == include_equal` → `false` → selects `cumulative(i)`. For `str`/`u64` the same expression still lives in code but the **order of cumulative-index selection** is inverted — it picks `cumulative(i)` when the condition is `false` where `i64`/`dbl` pick `cumulative(i-1)`. Review line 1299 filed this at an earlier commit; it survives at head.

8. **`numeric_domain_frac_i64_lt` divide-by-zero**: when adjacent buckets share the same `hi` (possible when MCV buckets are placed next to duplicate values), `(hi - lo) = 0` yields `inf/nan` from the `(double)(v-lo) / (double)(hi-lo)` computation. Only the dbl variant guards; i64, u64, dbl-dbl all at risk.

9. **`like_match_string` uses `.c_str()` on embedded-NUL bucket strings**. For BIT/VARBIT columns whose bytes include `\0`, `pattern.c_str()` truncates; the per-bucket match iterates `*s != '\0'` and terminates early. `.data()` + length-bound iteration is needed. LIKE selectivity on any BIT column with internal zero bytes is silently wrong.

10. **`_db_histogram` has no FK**. Orphan rows possible on crash between attribute-loop and class-drop. A reused OID can mis-attribute stale histogram data to a new class.

### Correctness — medium priority

11. **`DATETIMETZ`/`DATETIMELTZ` partial-write**. `is_histogrammable_type` accepts, `histogram_extract_key` handles, but `histogram_builder.cpp` `default`-falls and fails. `analyze_classes` writes `null_frequency` first, then builder fails — catalog row ends up with `null_frequency` populated and `histogram_values = NULL`. Reader sees "no histogram" via `histogram_blob_ptr == NULL` path; all good — until someone reads `null_frequency` alone and trusts it for partial predicates.

12. **`qo_between_selectivity` untouched** — plain `WHERE x BETWEEN 1 AND 10` gets 0.01 baseline; same query rewritten `WHERE x >= 1 AND x <= 10` gets histogram-based. Query parser canonicalizes most `BETWEEN`s into `PT_RANGE`, but not all paths.

13. **`PT_NOT_LIKE` bypasses `qo_not_selectivity`** — directly `1 - qo_like_selectivity(...)`. `qo_not_selectivity` may handle NULL semantics / clamping; bypassing changes behavior subtly.

14. **`PT_IN` (`qo_all_some_in_selectivity`) untouched** — ideal histogram candidate (per-element equal-selectivity sum). Default 0.01 persists.

15. **`selectivity_backup` dead local** (`qo_range_selectivity`) — declared, assigned, never read. Greptile P0 around iterator-reset; related but distinct.

16. **`PT_SHOW_HISTOGRAM` in `do_replicate_statement`** — SHOW is read-only, should never replicate. Slave's `la_apply_statement_log` correctly ignores, but the replication stream carries wasted entries.

17. **TRUNCATE does not invalidate histograms.** `schema_class_truncator.cpp` is only a constant relocation, no functional change. After `TRUNCATE`, bucket cumulatives refer to deleted rows; planner uses them. Until an explicit `ANALYZE … UPDATE HISTOGRAM`, plans are wrong on truncated-then-repopulated tables.

18. **No privilege check on histogram DDL.** All three `do_*_histogram` wrappers `AU_DISABLE`-wrap their entire body. Semantic check validates class existence, not ownership or grant-ALTER. Any user with `SELECT` can run `DROP HISTOGRAM` on any table they can see.

19. **`sm_drop_histogram` silent no-op on "not found".** `smt_check_histogram_exist` returning `NO_ERROR` means "absent"; `sm_drop_histogram`'s early return then yields success with nothing dropped. User expecting "no such histogram" error gets green-light. Inconsistent with `sm_drop_index` which raises `ER_SM_NO_INDEX`.

20. **Host-var-late-binding dependency is implicit.** Optimizer dereferences `env->parser->host_variables[index]` whenever `pc_rhs == PC_HOST_VAR`, with no bounds check and no PRM gate. User-visible as "histograms don't help unless you set `hostvar_late_binding=true`"; code-visible as a segfault risk if the DB_VALUE is corrupt.

### Code concerns / smells

21. **`classobj_check_histogram_exist` declared but not defined** (`class_object.h:1154`). Any future caller fails to link. Latent.

22. **`assert(oid_count < 2)` in `obj_find_multi_attr`** references a never-assigned local initialized to 0. Dead code remnant.

23. **`dbi_compat.h` alias inconsistency** — only `SQLX_CMD_DROP_HISTOGRAM` defined; UPDATE and SHOW missing. Breaks the file's 1:1 convention.

24. **Blob header documentation contradicts code.** Header comment says magic `HST2`, endianness LE, `f64 cumulative`/`approx_ndv`. Actual: `HST1`, big-endian (`OR_PUT_INT` is BE), `int64_t`. Future maintainers reading the spec first will corrupt the format.

25. **`db_make_varbit(..., 1073741823, blob, len*8)`** — magic number; use `SM_MAX_STRING_LENGTH`. Bit-length overflow possible if len > 128MB (theoretical).

26. **`print` functions lose information**. `pt_print_analyze_histogram` does not print `WITH n BUCKETS` or `WITH FULLSCAN`; malformed `on ( )` when `target_columns == NULL`. Plan-cache key stability at risk; any re-parse recovers default 300/false.

27. **Reserved-word strategy** — `HISTOGRAM`/`BUCKETS` fully reserved without `identifier:` fallback. Any existing schema with these names fails to parse. Fix: follow `HEAP`/`FULLSCAN` pattern.

28. **Display-name casing inconsistency** (`pt_show_node_type`): `"update_histogram"` vs `"show histogram"` vs `"DROP_HISTOGRAM"` — three conventions in three lines.

29. **Doxygen drift**: `pt_print_analyze_histogram` still carries `pt_print_update_histogram` header from pre-rename.

30. **Fixed RNG seed `123456789u`** — Greptile flagged; bias risk. Use `std::random_device{}()` XOR thread-id hash.

31. **Sample-size-agnostic selectivity.** A 0.2%-sampled histogram on a 100M-row table is treated with the same confidence as a full-scan histogram on a 1k-row table. No error band, no fallback on low-confidence buckets.

32. **`selectivity_backup` dead local** (already noted as #15).

33. **`QO_GET_HIST_STATS` no `self_allocated` branch** — parallels `QO_GET_CLASS_STATS` which does have one; transient/self-allocated class-info entries crash on `->smclass->histogram` deref.

34. **Parallel-array assumption** — `HIST_STATS::histogram[]` is accessed by index parallel to `ATTR_STATS::attr_stats[]`, via a separately-maintained `attr_hist_statsp_index` counter in `qo_get_attr_info`. Invariant unasserted; easy to break if ATTR_STATS is ever filtered.

35. **Hardcoded attribute names as string literals** across 6+ functions (`"class_of"`, `"key_attr"`, `"histogram_values"` etc.). No `CT_HISTOGRAM_KEY_ATTR_COLUMN` constants. Any column rename must find-and-replace.

36. **`update_histogram_for_all_classes` has misleading file placement** (`network_interface_cl.c`) — no network I/O; belongs in `schema_manager.c` or `execute_schema.c`.

37. **`smt_check_histogram_exist_and_delete` does double lookup** — `sm_drop_histogram` first calls `smt_check_histogram_exist`, then `smt_check_histogram_exist_and_delete` which re-does the same UNIQUE-index probe. Consolidable.

38. **Comment-code drift in `scan_manager.c`**: comment says "30% / max 5000 pages", code does "33% / max 10000 pages".

39. **Reflow noise**: large ranges of `execute_statement.c` are pure line-wrap reflows, obscuring semantic changes.

40. **ABI: `dbt_create_object_internal` signature change** breaks out-of-tree consumers; no deprecation shim.

### Performance / operational concerns

41. **`UPDATE STATISTICS` now O(classes × attrs)** due to blanket `update_histogram_for_all_classes`. No sysprm gate. Large schemas (hundreds of tables × tens of columns) will see UPDATE STATISTICS time balloon.

42. **No sample-size floor guard.** Tables with `total_rows < 1000` drive histogram decisions with essentially no statistical validity. No minimum table size to enable histograms.

43. **Planner cost-model recalibration missing.** `DEFAULT_EQUAL_SELECTIVITY=0.001` stayed; now ATTR=const regularly drops to `1/total_rows ≈ 1e-7` on 10M-row tables. Join-plan choices (hash vs nested-loop) flip based on what was a heuristic constant. No rebalancing of cost constants accompanying.

44. **`MIN_NDV` threshold in `stats_adjust_sampling_weight` doubled** (1000→2000) via `MAX_HEAP_SAMPLING_PAGES = 10000 × 20 / 100`. Affects all sampling-scan NDV correction, not just histogram paths — unrelated plans may shift on the same merge.

45. **Cache coherency across clients**: each client caches its `SM_CLASS::histogram` independently; no bump of `sm_bump_local_schema_version()` after histogram add/drop. Client A continues using stale histogram after client B drops it, until A's class is evicted for unrelated reasons.

46. **Per-PT_NAME 16-byte cost** (`histogram` pointer + `null_frequency` double) on every node, even for queries that will never consult histograms. Queries with large WHERE-clauses and many column references pay a parse-tree memory tax.

47. **Forced `ORDER BY` in `db_histogram` view** serializes every `SELECT * FROM db_histogram`. Comment in source hints author also noticed — `/* Is it possible to remove ORDER BY? */`.

## Baseline impact

- Before: `175442fc858bd0075165729756745be6f8928036`
- After: `175442fc858bd0075165729756745be6f8928036` (unchanged — PR is open)
- Bump triggered: `false`
- Logged: see [[log]] entry `[2026-04-24] pr-ingest PR #6753`.

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/6753
- Jira: [CBRD-26202](https://jira.cubrid.org/browse/CBRD-26202)
- Components touched: [[components/optimizer]], [[components/parser]], [[components/parse-tree]], [[components/semantic-check]], [[components/name-resolution]], [[components/execute-schema]], [[components/execute-statement]], [[components/scan-manager]], [[components/heap-file]], [[components/btree]], [[components/storage]], [[components/schema-manager]], [[components/object]], [[components/system-catalog]], [[components/dbi-compat]], [[components/system-parameter]], [[components/log-manager]], [[components/utility-binaries]], [[components/csql-shell]], [[components/base]]
- Sources: [[sources/cubrid-src-query]], [[sources/cubrid-src-object]], [[sources/cubrid-src-parser]], [[sources/cubrid-src-storage]], [[sources/cubrid-src-compat]]
- Adjacent PR: [[prs/PR-7062-parallel-scan-all-types]] (shares base commit, overlapping parser/xasl/optimizer edits — potential merge-time conflicts)
- Previous ingest: [[prs/PR-6911-parallel-heap-scan-io-bottleneck]]
