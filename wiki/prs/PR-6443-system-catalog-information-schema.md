---
type: pr
pr_number: 6443
pr_url: "https://github.com/CUBRID/cubrid/pull/6443"
repo: "CUBRID/cubrid"
state: MERGED
is_draft: false
author: "kangmin5505"
merged_at: "2025-12-12T07:52:33Z"
merge_commit: "1f12632170cf85f17170c25738c6bea7d6570373"
base_ref: "develop"
head_ref: "feature/system-catalog"
base_sha: "7636b4fe9bd6c3588ad3100dbe565a0ec36a58f3"
head_sha: "c40c58776a698369eb94c7a6e877591c7f506831"
jira: "CBRD-25862"
files_changed:
  - "msg/{en_US,ko_KR}.utf8/cubrid.msg"
  - "contrib/scripts/check_reserved.sql"
  - "src/storage/catalog_class.c (+488/-114, the largest)"
  - "src/storage/catalog_class.h (NEW FILE, +50)"
  - "src/storage/storage_common.h (+75)"
  - "src/storage/btree.h (comment fix)"
  - "src/storage/oid.{c,h}"
  - "src/storage/statistics_sr.c (+21/-5)"
  - "src/base/object_representation_sr.c (+39/-35)"
  - "src/base/error_code.h"
  - "src/object/schema_system_catalog_install.cpp (+468/-408)"
  - "src/object/schema_system_catalog_install.hpp"
  - "src/object/schema_system_catalog_install_query_spec.cpp (+165/-53)"
  - "src/object/schema_system_catalog.cpp"
  - "src/object/schema_system_catalog_builder.cpp"
  - "src/object/schema_system_catalog_constants.h"
  - "src/object/class_object.{c,h} (+167/-111 c, +33/-1 h)"
  - "src/object/transform.{c,h} (+16/-1 c, +60/-0 h)"
  - "src/object/transform_cl.c"
  - "src/object/schema_template.c (+57/-11)"
  - "src/object/schema_manager.{c,h} (+42/-49 c)"
  - "src/object/trigger_manager.{c,h} (+64/-170 c — net REMOVAL of 106 LOC)"
  - "src/object/object_primitive.c"
  - "src/object/work_space.c"
  - "src/object/authenticate.{c,h}"
  - "src/object/authenticate_access_class.cpp"
  - "src/object/authenticate_access_user.cpp (+67/-1)"
  - "src/object/authenticate_constants.h"
  - "src/object/authenticate_context.{cpp,hpp} (+75/-57 cpp)"
  - "src/object/authenticate_grant.cpp"
  - "src/object/authenticate_owner.cpp"
  - "src/object/authenticate_password.cpp"
  - "src/compat/db_obj.c (+55)"
  - "src/compat/db.h"
  - "src/compat/db_date.c"
  - "src/compat/db_temp.c"
  - "src/compat/dbtype_def.h"
  - "src/communication/network_interface_cl.c (+44/-214 — net REMOVAL of 170 LOC)"
  - "src/transaction/boot_cl.c (+38/-34)"
  - "src/transaction/boot_sr.c"
  - "src/transaction/locator_sr.c"
  - "src/transaction/log_applier.c"
  - "src/transaction/log_applier_sql_log.c"
  - "src/transaction/log_manager.c"
  - "src/transaction/log_tran_table.c"
  - "src/transaction/mvcc.c"
  - "src/sp/sp_catalog.{cpp,hpp} (+36/-3 cpp)"
  - "src/sp/sp_constants.hpp"
  - "src/sp/jsp_cl.cpp (+20/-1)"
  - "src/sp/pl_struct_compile.{cpp,hpp} (+82/-1 cpp, +17/-0 hpp)"
  - "pl_engine/pl_server/.../jsp/data/CompileInfo.java"
  - "pl_engine/pl_server/.../jsp/data/Dependency.java (NEW, +96)"
  - "pl_engine/pl_server/.../jsp/protocol/UnPackableObject.java"
  - "pl_engine/pl_server/.../plcsql/compiler/ParseTreeConverter.java"
  - "pl_engine/pl_server/.../plcsql/compiler/PlcsqlCompilerMain.java"
  - "pl_engine/pl_server/.../plcsql/compiler/serverapi/SqlSemantics.java"
  - "pl_engine/pl_server/.../plcsql/compiler/type/TypeRecord.java"
  - "pl_engine/pl_server/.../plcsql/compiler/visitor/TypeChecker.java (+19/-2)"
  - "src/query/serial.c"
  - "src/query/string_opfunc.{c,h} (+24/-1 c)"
  - "src/query/execute_statement.{c,h} (+104/-24 c)"
  - "src/query/execute_schema.c (+23/-1)"
  - "src/query/query_executor.c"
  - "src/parser/parse_tree.h"
  - "src/parser/parse_tree_cl.c"
  - "src/parser/parser_support.c"
  - "src/parser/semantic_check.c"
  - "src/parser/view_transform.c"
  - "src/optimizer/query_graph.c"
  - "src/executables/migrate.c"
  - "src/executables/unload_object.c"
  - "src/executables/unload_schema.c (+20/-16)"
  - "src/executables/util_sa.c"
  - "src/loaddb/load_server_loader.cpp"
related_components:
  - "[[components/system-catalog]]"
  - "[[components/object]]"
  - "[[components/schema-manager]]"
  - "[[components/storage]]"
  - "[[components/authenticate]]"
  - "[[components/sp]]"
  - "[[components/execute-statement]]"
  - "[[components/execute-schema]]"
  - "[[components/transaction]]"
  - "[[components/server-boot]]"
  - "[[components/utility-binaries]]"
  - "[[components/communication]]"
related_sources:
  - "[[sources/cubrid-src-storage]]"
  - "[[sources/cubrid-src-object]]"
  - "[[sources/cubrid-src-sp]]"
  - "[[sources/cubrid-src-compat]]"
ingest_case: b
triggered_baseline_bump: false
baseline_before: "175442fc858bd0075165729756745be6f8928036"
baseline_after: "175442fc858bd0075165729756745be6f8928036"
reconciliation_applied: false
reconciliation_applied_at:
incidental_enhancements_count: 1
tags:
  - pr
  - cubrid
  - system-catalog
  - information-schema
  - merged
  - refactor
  - db-authorizations-removal
  - naming-convention
created: 2026-04-26
updated: 2026-04-26
status: merged
---

# PR #6443 — Improve and refactor System Catalog for Information Schema

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `MERGED` · **Author:** `@kangmin5505` · **Jira:** [CBRD-25862](https://jira.cubrid.org/browse/CBRD-25862) (+ [CBRD-26036](https://jira.cubrid.org/browse/CBRD-26036) for view-side reflection)
> **Merge commit:** `1f12632170cf85f17170c25738c6bea7d6570373` · **Merged:** 2025-12-12 07:52:33 UTC
> **Base → Head:** `develop` (`7636b4fe`) → `feature/system-catalog` (`c40c5877`)
> **Scale:** 86 files, +2755 / −1527 (4282 LOC). Largest files: `catalog_class.c` (+488/−114), `schema_system_catalog_install.cpp` (+468/−408), `class_object.c` (+167/−111), `schema_system_catalog_install_query_spec.cpp` (+165/−53). Multi-axis Scale-rule trigger; analyzed via 5 parallel deep-read subagents.
> **Approvals:** @jongmin-won, @shparkcubrid, @beyondykk9, @hyunikn, @ctshim, @hornetmj, @airnet73 (7 approvals across Oct–Nov 2025).

> [!note] Ingest classification: case (b) — already absorbed in baseline
> Merge commit `1f12632170` is an ancestor of baseline `175442fc858b…`. Retroactive doc only — no Reconciliation Plan, no baseline bump. The changes ARE in baseline; this page documents what they were and what bugs they shipped with.

> [!warning] Rollup PR with 14+ sub-commits over `feature/system-catalog`
> The `gh pr diff` output is the rollup; the actual changes span sub-commits `f4f2d857c` (db_authorizations removal, May 2025), `09128e0e9` (naming convention), `b6b210dc3` (start_val), `8c063d7b8` (is_loginable / is_system_created), `789378dfa` (sql_data_access), `c1b0801fe` (class_partition_type), `2a61602a1` (_db_index attrs), `971caa5bf` (statistics_strategy), `bf1616799` (perf), `9f973f9e1` (revert is_system_class rename + add flags), `0b00be457` (checked_time NULL init), `4d67bb6dc` (META_CLASS revert), `d871bb7ff` (deadlock fix), `aaf8725a7` (change_serial_owner — partial), `b80679522` (php4 sub-ref revert), `565d8ed05` (trigger install relocation), `d15df1872` (extra timestamps), `563239c12` (auth catalog refactor). Anyone reading just the merge diff will miss the sub-commit narrative.

## Summary

Lays the groundwork for SQL-standard `INFORMATION_SCHEMA` views by adding metadata columns to existing system catalog classes, introducing a naming convention (`_db_*` for the storage class, `db_*` for the user-facing view), removing the legacy methods-only `db_authorizations` class, splitting the on-disk `is_system_class` flag bit into a separate `flags` column, adding three performance indexes on hot catalog joins, and threading dependency tracking through the PL/CSQL compile result. Eight existing system catalog classes gain new columns: `_db_class` (`statistics_strategy`, `flags`, `created_time`, `updated_time`, `checked_time`), `_db_index` (`referential_index`, `delete_rule`, `update_rule`, `referential_match_option`, `index_type`, `options`, `created_time`, `updated_time`), `_db_partition` (`class_partition_type`), `_db_serial` (`start_val`, `created_time`, `updated_time`), `_db_stored_procedure` (`sql_data_access`, `created_time`, `updated_time`), `_db_synonym`/`_db_server`/`_db_trigger` (timestamps). `db_user` separately gains `is_loginable` and `is_system_created` via the auth subsystem. `trigger_manager.c` loses `define_trigger_classes` (~140 LOC) — trigger schema definition is unified into `schema_system_catalog_install.cpp`. `network_interface_cl.c` `stats_update_all_statistics` is rewritten (−170 LOC) to use `locator_get_all_class_mops` instead of a hand-rolled SQL query.

## Motivation

CUBRID's `db_*` views were historically inconsistent — some columns were missing entirely, some were exposed as raw bitmask integers, system tables and views shared names. The Information Schema work needed:

1. **Per-row provenance metadata** — `created_time`, `updated_time`, `checked_time` columns to support audit trails and stale-statistics detection.
2. **Per-row policy metadata** — `is_loginable`, `is_system_created` on users; `sql_data_access` on stored procedures; `referential_match_option` / `delete_rule` / `update_rule` on indexes.
3. **Stable naming convention** — physical catalog class `_db_*`, view `db_*`. The old setup conflated names.
4. **Index coverage on join hot paths** — `_db_class.class_of`, `_db_data_type.(type_id, type_name)`, `_db_collation.coll_id` are referenced in every `db_*` view's authorization subquery.
5. **Dropping unused legacy** — `db_authorizations` was a methods-only "old root" class, never finished migrating to `db_root`.

## Changes

### Structural

#### New file
- `src/storage/catalog_class.h` (50 LOC) — extracts the public interface of `catalog_class.c` from elsewhere (was previously inline / in other headers).

#### New types in `src/storage/storage_common.h` (+75 LOC at 991-1097, plus 1 line at 1134)
- `enum SM_INDEX_FLAG { NONE=0, FILTER=1, FUNCTION=2, PREFIX=3 }` — replaces magic `0x01/0x02/0x03` ints in `or_install_btids_class` and `catcls_get_or_value_from_indexes`.
- `enum SM_CONSTRAINT_FIXED_FIELD_REVERSE_INDEX { COMMENT_INDEX=1, OPTIONS_INDEX=2, INDEX_TYPE_INDEX=3, STATUS_INDEX=4, OPTIONAL_INFO_INDEX=5, FIXED_FIELD_COUNT=5 }` — encodes reverse position of fixed fields at the tail of an `SM_CLASS_CONSTRAINT` sequence. New: `OPTIONS_INDEX` and `INDEX_TYPE_INDEX`.
- `enum SM_FK_INFO_*_INDEX` — six positions in the FK-info subsequence; new: `INDEX_CATALOG_OF_REF_CLASS_INDEX = 4`, `REF_MATCH_OPTION_INDEX = 5`.
- `enum SM_FOREIGN_KEY_MATCH_OPTION { SM_FK_MATCH_NONE = 0, PARTIAL = 1, FULL = 2 }`.
- `#define SERIAL_ATTR_START_VAL "start_val"`.
- Inline helpers `get_class_constraint_att_count(size)` and `get_class_constraint_index(size, reverse_idx)` — size-aware position math; replaces `seq_size - 2` / `seq_size - 3` magic offsets across all callers.

#### New/extended types in `src/object/transform.h` (+60/0)
- `enum CT_ATTR_CLASS_INDEX` (lines 91-123) — 30 ordinals indexing `ct_class_atts` slots (`CT_CLASS_CLASS_OF_INDEX = 0` through `CT_CLASS_PARTITION_INDEX`). New: `CT_CLASS_STATISTICS_STRATEGY_INDEX = 11`, `CT_CLASS_FLAGS_INDEX = 12`, `CT_CLASS_CREATED_TIME_INDEX = 13`, `CT_CLASS_UPDATED_TIME_INDEX = 14`, `CT_CLASS_CHECKED_TIME_INDEX = 15`.
- `enum CT_ATTR_INDEX_INDEX` (lines 125-147) — 20 ordinals indexing `ct_index_atts`. New: `CT_INDEX_REFERENTIAL_INDEX_INDEX = 11`, `…DELETE_RULE_INDEX`, `…UPDATE_RULE_INDEX`, `…REFERENTIAL_MATCH_OPTION_INDEX`, `…INDEX_TYPE_INDEX`, `…OPTIONS_INDEX`, `…CREATED_TIME_INDEX = 18`, `…UPDATED_TIME_INDEX = 19`.
- TODO: "create CT_ATTR_*_INDEX of other CT_CLASSes" — left for later.

#### New columns per system catalog class

| Class | New columns (type) | Schema source |
|---|---|---|
| `_db_class` | `statistics_strategy` int, `flags` int (split from `is_system_class`), `created_time` datetime, `updated_time` datetime, `checked_time` datetime | `schema_system_catalog_install.cpp:436-452` |
| `_db_index` | `referential_index` object→CT_INDEX (self-FK), `delete_rule` int, `update_rule` int, `referential_match_option` int, `index_type` int, `options` int, `created_time` datetime, `updated_time` datetime | `schema_system_catalog_install.cpp:712-720` |
| `_db_partition` | `class_partition_type` int (renamed from "depth" before merge — values 0/1/2 = `DB_NOT_PARTITIONED_CLASS`/`DB_PARTITIONED_CLASS`/`DB_PARTITION_CLASS`) | `schema_system_catalog_install.cpp:848` |
| `_db_serial` | `start_val` numeric(38,0) default `1`, `created_time` datetime, `updated_time` datetime | `schema_system_catalog_install.cpp:1009-1018` |
| `_db_stored_procedure` | `sql_data_access` int, `created_time`, `updated_time` | `schema_system_catalog_install.cpp:911-914` |
| `_db_synonym`, `_db_server`, `_db_trigger` | `created_time`, `updated_time` | install lines 820-821, 1191-1192, 1232-1233 |
| `db_user` (NOT `_db_user` — see naming exception below) | `is_loginable` int, `is_system_created` int | `authenticate_context.cpp:369-370` |

All time columns are `DB_TYPE_DATETIME`. Exception: `_db_stored_procedure_code.created_time` is `format_varchar(16)` (smell — see below).

#### New constraints / indexes

| Class | Constraint | Type | Site |
|---|---|---|---|
| `_db_class` | `INDEX (class_of)` (non-UNIQUE — see warning) | general index | `schema_system_catalog_install.cpp:478` |
| `_db_data_type` | `PRIMARY KEY (type_id, type_name)` | composite PK | `schema_system_catalog_install.cpp:872-878` |
| `_db_collation` | `PRIMARY KEY (coll_id)` | single-col PK | `schema_system_catalog_install.cpp:1112-1114` |

> [!warning] Why `_db_class.class_of` is not UNIQUE
> Comment at `schema_system_catalog_install.cpp:455-475` documents an `assert(false)` in `btree_key_insert_new_key` that fires under `RENAME CLASS` if a unique constraint exists on this column. The PR adds a general `DB_CONSTRAINT_INDEX` instead — gives the auth-subquery a B-tree without violating the rename invariant. The fix is documented as deferred.

#### New error code
- `ER_AU_LOGIN_DISABLED = -1369` (`error_code.h:1756`) — raised by `perform_login` when `is_loginable_user()` returns false.

#### New `cubthread::entry` / `SM_CLASS` field touchpoints
- `SM_PARTITION::class_partition_type DB_CLASS_PARTITION_TYPE` (`class_object.h:733`).
- `TR_TRIGGER::created_time / updated_time DB_DATETIME` (`trigger_manager.h:96-97`).
- **`SM_CLASS::flags` was NOT modified** — pre-existed (`class_object.h:800`). The PR's "split is_system_class into flags" refers exclusively to the `_db_class` catalog row split.

#### New helpers in `src/compat/db_obj.c` (+55)
- `db_set_otmpl_timestamps(DB_OTMPL *)` — both `created_time` + `updated_time` on a template.
- `db_update_otmpl_timestamp(DB_OTMPL *)` — `updated_time` only on a template.
- `db_update_obj_timestamp(DB_OBJECT *)` — `updated_time` on a live MOP via `db_put`.

All declared in `db.h:286-288`.

#### Public-API touch
- `dbtype_def.h:494` — comment-only: `DB_OBJECT_SERIAL = 2, /* SERIAL (db_serial) */` → `(_db_serial)`. Reflects the rename.

#### Public-API ABI
- No new types in `dbtype_def.h`/`db.h` beyond the three timestamp helpers. Enum order preserved. ABI-safe.

#### Files removed (entire functions)
- `trigger_manager.c::define_trigger_classes()` (~140 LOC) — trigger class definition migrates to `schema_system_catalog_install.cpp:276` (`ADD_TABLE_DEFINITION (CT_TRIGGER_NAME, system_catalog_initializer::get_trigger())`).
- `trigger_manager.c::tr_install()` and its header decl (`trigger_manager.h:260`).
- `schema_manager.c::sm_mark_system_classes()` — bulk post-init scan replaced by per-class flag-set at install time.
- `db_authorizations` — methods-only legacy class deleted by sub-commit `f4f2d857c` (#6039 from May 2025).

### Per-file notes

- `src/storage/catalog_class.c` (+488/−114) — server-side bridge between heap-class records and `_db_*` catalog rows. New helpers: `catcls_set_or_value_timestamps` (4063), `catcls_update_or_value_updated_time` (4125), `catcls_update_or_value_class_stats_fields` (4133), `catcls_copy_or_value_times_and_statistics` (4089), `catcls_update_class_stats` (4452). New global var arrays `_gv_ct_Class_*_idx` and `_gv_ct_Index_*_idx` resolved at boot by `catcls_cache_fixed_attr_indexes` (4646-4729). Flag-split logic at 1055-1060 ([[components/system-catalog]]).
- `src/storage/catalog_class.h` (NEW, +50) — public prototypes ([[components/system-catalog]]).
- `src/storage/storage_common.h` (+75) — new enums (above) ([[components/storage]]).
- `src/storage/statistics_sr.c` (+21/-5) — `xstats_update_statistics` now calls `catcls_update_class_stats` (passes class_name + ci_time_stamp + with_fullscan) at line 316 (and 1388 for partitioned) ([[components/storage]]).
- `src/base/object_representation_sr.c` (+39/-35) — `or_install_btids_constraint` assert hardened from `>= 8` to `>= 10` (line 2262); `or_install_btids_class` migrates from magic offsets to `get_class_constraint_att_count` / `get_class_constraint_index`; magic flag ints renamed to `SM_INDEX_FLAG_*` ([[components/storage]]).
- `src/storage/btree.h` — comment-only `db_serial` → `_db_serial` ([[components/btree]]).
- `src/storage/oid.{c,h}` — REMOVED `OID_CACHE_OLD_ROOT_CLASS_ID` and `oid_Authorizations_class` cache slot. Renamed `CT_DB_SERVER_NAME` → `CT_SERVER_NAME` in `oid_Cache[]` ([[components/storage]]).
- `src/object/schema_system_catalog_install.cpp` (+468/-408) — declarative class+view definitions; trigger class registration moves here ([[components/system-catalog]]).
- `src/object/schema_system_catalog_install_query_spec.cpp` (+165/-53) — view SQL bodies; new projections for `statistics_strategy`, `is_reuse_oid_class`, `referential_index*`, `delete_rule`/`update_rule`/`referential_match_option`/`index_type`, `class_partition_type`, `start_val`, time columns. `db_stored_procedure.sql_data_access` is **commented-out** with a `TODO: implement sql_data_access` (lines 1233-1240) ([[components/system-catalog]]).
- `src/object/schema_system_catalog.cpp` — minor; removes `db_authorizations` registration.
- `src/object/schema_system_catalog_constants.h` — physical class renames: `CT_SERIAL_NAME`: `db_serial` → `_db_serial`, `CT_HA_APPLY_INFO_NAME`: `db_ha_apply_info` → `_db_ha_apply_info`, `CT_TRIGGER_NAME`: `db_trigger` → `_db_trigger`. View renames: `CTV_TRIGGER_NAME`: `db_trig` → `db_trigger` (with new views `CTV_SERIAL_NAME = "db_serial"` and `CTV_HA_APPLY_INFO_NAME` taking the freed-up names). `db_user`/`db_root`/`db_password`/`db_authorization` keep their `db_*` prefix (auth-class exception).
- `src/object/class_object.{c,h}` (+167/-111 c, +33/-1 h) — extends FK info schema with `index_catalog_of_ref_class` MOP + `ref_match_option`; extends constraint property-seq layout with `index_type` + `options`; new `SM_INDEX_FLAG` enum usage; new `SM_PARTITION::class_partition_type` field ([[components/object]]).
- `src/object/transform.{c,h}` (+76 net) — new `CT_ATTR_CLASS_INDEX` and `CT_ATTR_INDEX_INDEX` enums; extends `ct_class_atts` (+5), `ct_index_atts` (+9 with reorder), `ct_partition_atts` (+1), `partition_atts` (+1) ([[components/object]]).
- `src/object/transform_cl.c` — partition writer/reader handle `class_partition_type` (lines 4900, 4986); removes an `assert(vars != NULL)` (potentially fragile — see Smells) ([[components/object]]).
- `src/object/schema_template.c` (+57/-11) — new helper `find_index_catalog_class` (4926-4949); `smt_add_constraint_to_property` gains `int options` parameter; `smt_add_constraint` extracts deduplicate level via `OPTION_DEDUPLICATE_MASK` ([[components/schema-manager]]).
- `src/object/schema_manager.{c,h}` (+42/-49 c) — REMOVES `sm_mark_system_classes`; adds AU_DISABLE harness around serial rename/delete in `sm_rename_class` and `sm_delete_class_mop`; `sm_update_statistics` cleanup with non-cast bool propagation; rebuilt `sm_update_all_catalog_statistics` class list ([[components/schema-manager]]).
- `src/object/trigger_manager.{c,h}` (+64/-170 c) — REMOVES `define_trigger_classes` (~140 LOC), unified into `schema_system_catalog_install.cpp:276`; new fields/helpers `tr_set_trigger_timestamps`, `tr_update_trigger_timestamp`; new attrs `TR_ATT_CREATED_TIME`/`TR_ATT_UPDATED_TIME` ([[components/object]]).
- `src/object/object_primitive.c` (+9/-2) — `mr_data_readval_datetime` recognizes `DATETIME_IS_NULL` sentinel `{UINT_MAX, UINT_MAX}` and returns NULL DB_VALUE; required for back-compat with rows lacking the new datetime columns ([[components/db-value]]).
- `src/object/work_space.c` (+1/-1) — comment-only `db_serial` → `_db_serial` ([[components/object]]).
- `src/object/authenticate_context.{cpp,hpp}` (+75/-57 cpp) — installs `is_loginable`/`is_system_created` on `db_user`; new methods `set_system_user`, `disable_login` (UNUSED — see Smells), `is_loginable_user`; `perform_login` gates on `is_loginable_user` returning ER_AU_LOGIN_DISABLED; renames internal `authorizations_class` field → `root_class` ([[components/authenticate]]).
- `src/object/authenticate_access_user.cpp` (+67/-1) — `au_make_user` defaults: `is_loginable=true`, `is_system_created=false`; new `au_set_user_timestamps` and `au_update_user_timestamp` helpers; `updated_time` bumps inserted in `au_add_member_internal`, `au_drop_member`, `au_drop_user` group reflow ([[components/authenticate]]).
- `src/object/authenticate_access_class.cpp` — single-line `Au_authorizations_class` → `Au_root_class` rename in `is_protected_class`. The check is now BROADER, not narrower — it correctly protects the actual `db_root` MOP instead of the legacy alias ([[components/authenticate]]).
- `src/communication/network_interface_cl.c` (+44/-214) — REWRITES `stats_update_all_statistics` from a hand-rolled SQL union over `_db_class` to `locator_get_all_class_mops` + `is_top_level_class` filter loop. Unifies CS/SA paths ([[components/communication]]).
- `src/transaction/boot_cl.c` (+38/-34) — extracts new helper `install_system_metadata` consolidating the `au_init/install/start + tr_init + catcls_init/install` sequence at line 202-230. Pure refactor ([[components/server-boot]]).
- `src/sp/sp_constants.hpp` — defines `SP_SQL_DATA_ACCESS_TYPE` enum (lines 149-156): `UNKNOWN=-1, NO_SQL, CONTAINS_SQL, READS_SQL_DATA, MODIFIES_SQL_DATA` ([[components/sp]]).
- `src/sp/sp_catalog.{cpp,hpp}` (+36/-3 cpp) — `sp_info::sql_data_access` field, default `UNKNOWN`; written at line 546-547. **Never set by parser** — see Smells ([[components/sp]]).
- `src/sp/pl_struct_compile.{cpp,hpp}` (+82/-1 cpp, +17 hpp) — new `plcsql_dependency` struct mirroring Java side; added to `compile_response::dependencies` and `sql_semantics::dependencies` vectors. Pack/unpack methods. **Wire format changed without protocol-version bump** — see Smells ([[components/sp]]).
- `pl_engine/pl_server/.../jsp/data/Dependency.java` (NEW, +96) — Java mirror with 7 `OBJ_TYPE_*` constants (TABLE/VIEW/FUNCTION/PROCEDURE/SERIAL/TRIGGER/SYNONYM). Final, immutable, packable. TODO: "use server-defined enumeration" — magic ints duplicated server-side. `OBJ_TYPE_VIEW`, `OBJ_TYPE_TRIGGER`, `OBJ_TYPE_SYNONYM` declared but never emitted by `TypeChecker.java` ([[components/sp]]).
- `pl_engine/pl_server/.../plcsql/compiler/visitor/TypeChecker.java` (+19/-2) — emits `Dependency` records during AST walk: TABLE for `%ROWTYPE`/`%TYPE` refs (101, 109), FUNCTION for global function call (553), SERIAL for `seq.NEXTVAL`/`seq.CURRVAL` (817), PROCEDURE for procedure call (1231) ([[components/sp]]).
- `src/query/execute_statement.c` (+104/-24) — `do_create_serial_internal` writes `start_val` at 770-775 (immutable post-create). `do_drop_serial` requires `AU_DISABLE` wrap + reimplemented `au_check_serial_authorization` because `_db_serial` is now DBA-owned. **Anti-pattern**: many catalog writes now AU_DISABLE then re-implement auth checks in app code (lost defense-in-depth) ([[components/execute-statement]]).
- `src/query/execute_schema.c` (+23/-1) — `class_partition_type` write at 15434-15443 in partition node walker ([[components/execute-schema]]).
- `src/executables/unload_schema.c` (+20/-16) — `AU_DISABLE` wraps added before every `db_compile_and_execute_local` query against `_db_serial`/`_db_user`. **`start_val` NOT included in unload SELECT** (lines 704-711) — round-trip-lossy ([[components/utility-binaries]]).
- `src/executables/migrate.c` — **NOT extended.** No in-place upgrade path for any of the new columns ([[components/utility-binaries]]).

### Behavioral

1. **`flags` column on `_db_class` is split from `is_system_class` at the catalog-row layer only.** The on-disk heap-class record's single `flags` int (offset `ORC_CLASS_FLAGS = 64`) is unchanged. `catalog_class.c:1055-1060` writes:
   ```c
   db_make_int (&attrs[CT_CLASS_IS_SYSTEM_CLASS_INDEX].value, flags & SM_CLASSFLAG_SYSTEM);
   db_make_int (&attrs[CT_CLASS_FLAGS_INDEX].value,           flags & ~SM_CLASSFLAG_SYSTEM);
   ```
   So `_db_class.is_system_class` carries bit 0 only (legacy semantic), and `_db_class.flags` carries bits 1+ (`WITHCHECKOPTION`, `LOCALCHECKOPTION`, `REUSE_OID`, `SUPPLEMENTAL_LOG`). The view `db_class.is_reuse_oid_class` is computed via `[c].[flags] & 8` (the `SM_CLASSFLAG_REUSE_OID` bitmask, hard-coded as the integer literal `8` in the view spec).
2. **Naming convention** — system catalog physical classes get `_db_*` prefix; views over them get `db_*`. Three classes were renamed in this PR: `db_serial` → `_db_serial`, `db_trigger` → `_db_trigger`, `db_ha_apply_info` → `_db_ha_apply_info`. Three new views with the freed-up names take their place. **Auth classes (`db_user`, `db_root`, `db_password`, `db_authorization`) are exempted** from the rename — they keep `db_*`. PR description mentions a `db_user` rename to `_db_user` that did NOT happen — `CT_USER_NAME` is still `"db_user"` (`schema_system_catalog_constants.h:48`).
3. **`db_authorizations` removed entirely** (sub-commit `f4f2d857c` from May 2025). Was a methods-only "old root" class with `add_user`, `drop_user`, `find_user`, `print_authorizations`, `info`, `change_owner`, `change_trigger_owner`, `get_owner` class methods — duplicating `db_root`. No replacement view. **No migration**: pre-existing databases keep an orphaned `_db_class` row pointing to a non-existent vclass spec.
4. **`is_loginable` defaults to true for new users; `is_system_created` defaults to false.** DBA and PUBLIC are marked `is_system_created=true` by `set_system_user()` but **NOT** `is_loginable=false` — so PUBLIC remains loginable post-install.
5. **`perform_login` now consults `is_loginable_user()`.** Returns `ER_AU_LOGIN_DISABLED` if false. **Critical:** if `obj_get` errors (e.g. transient lock conflict or missing column on un-migrated DB), `is_loginable_user` returns false silently → every login attempt fails with the wrong error code.
6. **`sql_data_access` column is reserved-but-broken.** Column declared on `_db_stored_procedure`, written on every `CREATE PROCEDURE` with the default `SP_SQL_TYPE_UNKNOWN = -1`. No parser path sets it; the corresponding view column is commented out. Reserved storage; user-facing surface zero.
7. **`start_val` is immutable post-create.** Written once during `do_create_serial_internal`; `ALTER SERIAL` and `ALTER TABLE … AUTO_INCREMENT = N` only touch `current_val`. `do_reset_auto_increment_serial` resets `current_val` to **MIN_VAL**, not `start_val` — variable misleadingly named `start_value` (`execute_statement.c:1029`).
8. **`unloaddb` does NOT include `start_val` in its `_db_serial` SELECT.** Round-trip via unload/reload zeros out the user-supplied `start_val` to default `1`.
9. **`db_partition.class_partition_type` view encoding loses information.** CASE only emits `'PARTITION CLASS'` for value `2`; values `0` (DB_NOT_PARTITIONED_CLASS) and `1` (DB_PARTITIONED_CLASS) produce NULL. Root partitioned tables and non-partitioned tables look identical to a `db_partition` reader.
10. **`network_interface_cl.c::stats_update_all_statistics` rewritten.** Pre-PR ran a hand-rolled SQL union over `_db_class` to enumerate top-level classes, then per-class fetched NDV via query, packed a request, and shipped `NET_SERVER_QST_UPDATE_STATISTICS`. Post-PR: `locator_get_all_class_mops(DB_FETCH_READ, is_top_level_class)` + per-class call. Unifies CS/SA paths. **Note:** `is_top_level_class` filter uses the new `class_partition_type` to identify partition-root vs leaf — leveraging the just-added column.
11. **Authorization defense-in-depth lost.** Every catalog write that used to ride on the user's grant now `AU_DISABLE`s and re-implements auth checks in C (e.g. `do_drop_serial` calls `au_check_serial_authorization` ad hoc). New paths must remember the check; nothing enforces it centrally.
12. **`SP_ATTR_SQL_DATA_ACCESS` view exposure deferred.** `db_stored_procedure` view declares the placeholder column and has `// TODO: implement sql_data_access` blocks at both the install and query-spec level.
13. **PL/CSQL dependency tracking added end-to-end.** `TypeChecker.java` walks the PL/CSQL AST, emitting `Dependency` records for table refs (via `%TYPE`/`%ROWTYPE`), function calls, procedure calls, and serial usages. Server-side parser supplements via `SqlSemantics.dependencies` for static SQL blocks. Aggregated into `CompileInfo` and shipped back to C-side `compile_response::dependencies`. **Consumer of these deps not in baseline** — the server-side reader at `175442fc` doesn't yet do anything with them (see Open Issues).

### New surface (no existing wiki reference)

- `src/storage/catalog_class.{c,h}` — server-side catalog-row maintenance layer. Not yet documented as its own page; mentioned only in `components/storage.md`.
- `CT_ATTR_CLASS_INDEX` / `CT_ATTR_INDEX_INDEX` enums — new public position-enums in `transform.h`.
- `SM_INDEX_FLAG`, `SM_FOREIGN_KEY_MATCH_OPTION`, `SM_CONSTRAINT_FIXED_FIELD_REVERSE_INDEX` — three new enums in `storage_common.h`.
- `Dependency.java` — new file in PL/CSQL compile path.
- `SP_SQL_DATA_ACCESS_TYPE` enum.

## Review discussion highlights

7 approvals; 4 inline comments are mostly trivial (`reviewdog` style nit, three author-self annotations explaining `AU_DISABLE` placements):

- **`@kangmin5505 @ schema_template.c:4936`** — "_db_index(system table) 의 값을 가져오기 위한 권한 권한 검사 해제 (신규 함수)". Author's own annotation that the new `find_index_catalog_class` helper deliberately disables auth to read `_db_index`.
- **`@kangmin5505 @ execute_statement.c:3015`** — "_db_serial(system table)에서 값을 가져오기 위한 권한 검사 해제 (기존: 일반 유저도 serial system table 대한 SELECT 권한이 있었음)". Confirms that pre-PR, normal users had implicit SELECT on `_db_serial`; post-PR, the AU_DISABLE wrap is the new pattern.
- **`@kangmin5505 @ unload_schema.c:744`** — same annotation for `unloaddb`.

No design-rationale debates in inline review. Most decisions were made on the long-lived `feature/system-catalog` branch and merged into the rollup PR after team review.

## Reconciliation Plan

n/a — case (b) absorbed.

## Pages Reconciled

n/a — case (b).

## Incidental wiki enhancements

1. **[[components/system-catalog]]** — extensive update: documented the naming convention (system catalog class `_db_*`, view `db_*`); listed the catalog-class renames (`db_serial`, `db_trigger`, `db_ha_apply_info`); documented `db_authorizations` removal; added the new index strategy (`_db_class.class_of` non-UNIQUE INDEX with rationale, `_db_data_type.(type_id, type_name)` composite PK, `_db_collation.coll_id` PK); referenced the new `CT_ATTR_*_INDEX` position enums in `transform.h`; added the on-disk `flags` split rule (`bit 0 → is_system_class`, `bits 1+ → flags`); added a "Catalog row provenance" section covering the per-row `created_time`/`updated_time`/`checked_time` columns and the `catcls_set_or_value_timestamps` write path.

## Deep analysis — supplementary findings

Synthesized from 5 parallel subagent reports. Bugs grouped by severity. Most are baseline-truths discovered during deep-read; several warrant follow-up correctness PRs.

### Correctness — high priority

1. **No migration path for any of the new catalog columns.** `migrate.c` was not extended. `boot_sr.c` does not have an upgrade hook. `catcls_cache_fixed_attr_indexes` (`catalog_class.c:4691-4694, 4721-4722`) has hard `assert(_gv_… != -1)` calls. **An old DB lacking `statistics_strategy` / `checked_time` / `flags` / `created_time` / `updated_time` will trip these on first boot under the new binary and SIGABRT.** Combined with finding 2, this means upgrade-without-dump-reload is impossible *and* dump-reload is lossy.
2. **`unload_schema.c` SELECT against `_db_serial` does NOT include `start_val`** (lines 704-711). Combined with finding 1, a round-trip `unloaddb`/`loaddb` cycle silently zeros out every user-specified `start_val` to default `1`. The `SERIAL_VALUE_INDEX` enum at the top of the file was not extended either — adding `start_val` here is a one-line follow-up.
3. **Login fails on un-migrated DBs.** `perform_login` (`authenticate_context.cpp:597`) calls `is_loginable_user()` which calls `obj_get(user, "is_loginable", &value)`. On an old DB lacking the column, `obj_get` returns an error and `is_loginable_user` returns `false` → every login attempt fails with `ER_AU_LOGIN_DISABLED`. Hard regression for any unmigrated database.
4. **`register_user_trigger` and `unregister_user_trigger` have INVERTED if-error logic** (`trigger_manager.c:1968-1971, 2036-2039`):
   ```c
   error = set_insert_element (table, 0, &value);
   if (error != NO_ERROR) {
       error = au_update_user_timestamp (Au_user);  // WRONG: updates only on FAILURE
   }
   ```
   The user-timestamp update fires only on failure, AND overwrites the original error code with the timestamp helper's return. **Silently breaks user-cache timestamping AND swallows real errors.** Same defect duplicated in both functions. High severity.
5. **`smt_add_constraint_to_property` has uninitialized `current_datetime`** (`schema_template.c:1550, 1564-1568, 1595`). `db_sys_datetime` is called but the result is never used; on the early `goto end` path (`classobj_find_prop_constraint` returns true at `:1558`), `db_sys_datetime` never runs, then `pr_clear_value (&current_datetime)` operates on uninitialized memory — undefined behavior.
6. **`catcls_set_or_value_timestamps` ignores `db_sys_datetime` failure** (`catalog_class.c:4081`). No return-code check, then dereferences `db_get_datetime(&datetime_val)`. If `db_sys_datetime` fails, `datetime_val` stays uninitialized → `db_get_datetime` returns garbage or NULL → `db_make_datetime` either crashes or stores garbage. Same pattern in `catcls_update_or_value_updated_time` (line 4129).
7. **`catcls_copy_or_value_times_and_statistics`** (`catalog_class.c:4099-4106`) preserves `checked_time` and `statistics_strategy` only when **both** are non-NULL. If only one is NULL (possible during rolling upgrade where stats ran but the column wasn't yet populated), neither survives the next update. Should be independent checks.
8. **`CONST op ATTR`-style bug class flagged**: `do_reset_auto_increment_serial` resets `current_val` to `MIN_VAL` rather than `start_val` (`execute_statement.c:1029` — variable misleadingly named `start_value` but value is `MIN_VAL`). The PR added `start_val` but did not refactor reset to use it. Possibly intentional for back-compat (TRUNCATE behavior); document or fix.
9. **Wire format extended without version handshake.** `compile_response` and `sql_semantics` packers (`pl_struct_compile.cpp`) gained trailing `dependencies` field with no protocol-version bump. Old C/Java pair will misalign — a JSP server speaking the old protocol against a new C-side will read the deps vector as some other field, with unpredictable corruption.

### Correctness — medium priority

10. **`disable_login()` is dead code** — declared private in `authenticate_context.hpp:196`, defined at `authenticate_context.cpp:974-989`, **zero callers** tree-wide. Hook for future Information Schema work.
11. **`is_loginable_user()` lossy boolean check** (`authenticate_context.cpp:995-1001`) — any error from `obj_get` is silently treated as "not loginable". Transient lock conflict on `_db_user` during login → `ER_AU_LOGIN_DISABLED` (wrong error, masks real cause).
12. **`au_set_user_comment` does NOT call `au_update_user_timestamp`** (`authenticate_access_user.cpp:680-719`). Comment changes silently leave `updated_time` stale. Inconsistent with every other state-change site that got the bump.
13. **`db_partition.class_partition_type` view loses info for value 1** (`schema_system_catalog_install_query_spec.cpp:1124`). Should label both root and leaf, or expose the raw int.
14. **`db_stored_procedure.sql_data_access` not exposed** (TODO in install + query_spec). Column is commented out in the view; users querying `db_stored_procedure` see nothing for `sql_data_access` even when set.
15. **`SP_SQL_DATA_ACCESS_TYPE` written as garbage on every `CREATE PROCEDURE`.** No parser path sets `sp_info::sql_data_access`; default `UNKNOWN = -1` is what gets written. Reserved-but-broken column.
16. **`OBJ_TYPE_VIEW` / `OBJ_TYPE_TRIGGER` / `OBJ_TYPE_SYNONYM`** declared in `Dependency.java` but never emitted by `TypeChecker.java`. Reserved enum values with no producer.
17. **No central enum for `OBJ_TYPE_*`.** Java declares 7 magic ints; C++ `plcsql_dependency::obj_type` is also untyped int. Both with `TODO: use predefined enum`. Wire-fragile — reordering will silently desynchronize ends.
18. **`SM_FOREIGN_KEY_RESTRICT` and `SM_FOREIGN_KEY_NO_ACTION` are distinct enum codes** but ISO SQL semantics differ by deferred-vs-immediate. Verify `do_check_fk_constraints` actually distinguishes them; otherwise they're aliases.
19. **`catalog_class.c:1057` comment ("split legacy packed flags ... recombine on write") describes nonexistent code.** No recombine path exists in `catcls_put_or_value_into_buffer`. The mechanism works because writes come from the heap-class transform (which produces the merged int already) — but the comment is misleading.
20. **`assert(vars != NULL)` removed in `disk_to_partition_info`** (`transform_cl.c:4969`). Subsequent code dereferences `vars[ORC_PARTITION_NAME_INDEX].length` without guard. Pre-PR assert at least caught it in debug. Removed without justification.
21. **`tr_drop_trigger_internal` removed `has_savepoint` flag** but the savepoint is still established via `tran_system_savepoint`. Without the flag, the function can no longer decide whether to rollback the savepoint on error — possible behavioral regression.

### Code concerns / smells

22. **`SM_CLASSFLAG_SYSTEM` is `#define`d in two places** — `class_object.h:307` and `catalog_class.c:67-68` (with comment "Keep in sync with the shared definition"). Two-place truth invites drift. Should be a shared header include.
23. **`_db_stored_procedure_code.created_time` is `format_varchar(16)`** (`schema_system_catalog_install.cpp:971`) — a 16-char string, not a datetime. Inconsistent with every other `created_time` in the catalog. Likely a leftover from when the SP code subsystem stored a stringified version hash; should be migrated or renamed.
24. **`change_serial_owner` CLASS_METHOD still registered** (`schema_system_catalog_install.cpp:1014`) despite sub-PR `aaf8725a7`'s claim "Remove change_serial_owner method". Either the rename to `_db_serial` was the actual scope (removed from public-name `db_serial`, kept on the underscore catalog), or removal is incomplete.
25. **Empty `_db_charset` constraint set** (`schema_system_catalog_install.cpp:1138`) — `charset_id` is a join target for `db_collation` and `db_charset` but has no PK or index. Inconsistent with the new `_db_collation.coll_id` PK.
26. **`is_reuse_oid_class` view column hard-codes the bitmask integer** via `sprintf` (`schema_system_catalog_install_query_spec.cpp:85, 137`) — if `SM_CLASSFLAG_REUSE_OID` ever changes value, the view stays correct only if the database is reinstalled. No test guards this invariant.
27. **`_db_class` no-PK warning is buried in a 20-line code comment** (`schema_system_catalog_install.cpp:455-475`). Future contributors are likely to "fix" the missing PK without reading it.
28. **Hardcoded magic `CATCLS_USER_ATTR_IDX_NAME = 11`** at `src/loaddb/load_server_loader.cpp:186`. Comment at `authenticate_context.cpp:359-361` acknowledges fragility ("If the attribute configuration is changed, the CATCLS_USER_ATTR_IDX_NAME also be changed"). The PR added two attributes before `comment` in `db_user`. Should be replaced with name-based lookup at boot time (similar to `catcls_cache_fixed_attr_indexes`).
29. **Constraint sequence assert hardening** (`object_representation_sr.c:2262`: `>= 8` → `>= 10`) silently turns all old-DB constraint sequences into hard SIGABRT. Same upgrade-without-migration risk as #1.
30. **`SM_INDEX_TYPE` enum has a single value with trailing comma** (`class_object.h:534-537`: `typedef enum { SM_BTREE_TYPE, } SM_INDEX_TYPE;`). Storing an integer column for a constant is wasteful but future-proofs for hash/spatial indexes.
31. **`is_loginable` and `is_system_created` stored as `integer`** (4 bytes each per user) instead of `short`. CUBRID has no native `bool` SQL type but `short` would be more compact.
32. **Boolean-as-integer naming for `Au_*` macros** — `set_system_user` uses `Au_dba_user`/`Au_public_user` macros instead of member fields, even though it's a method on `authenticate_context`. Self-referential macro hop on a member function.
33. **`AU_DISABLE` flag-based cleanup pattern** (`bool au_disable_flag = false; ... AU_DISABLE(save); au_disable_flag = true; ... if (au_disable_flag) AU_ENABLE(save);`) is repeated across many catalog write paths. RAII-style scope guard would be safer; this is C so it requires manual discipline.
34. **`tr_set_trigger_timestamps` returns `ER_FAILED` on `db_sys_datetime` failure** without `er_set` — caller's `goto error` runs with whatever `er_errid()` carries (usually fine but fragile).
35. **`db_set_otmpl_timestamps` and `db_update_otmpl_timestamp` duplicate the `db_sys_datetime` setup** in `db_obj.c:1942-1976`. Could share a helper.
36. **`au_check_serial_authorization` is not a centralized decorator** — every new DDL path on `_db_serial` must remember to call it after `AU_DISABLE`. Easy to forget.

### Performance

37. **`find_index_catalog_class` issues an extra server round-trip per FK column** (`schema_template.c:4926-4949`). DDL transaction has the ref class write-locked but `db_find_unique` issues a fresh server lookup. Performance hit on multi-column FKs.
38. **PR-rollup boundary problem**: `gh pr diff` for #6443 shows only the rollup diff (+2755/-1527), but cumulative churn on `schema_system_catalog_install.cpp` alone across 14+ sub-commits is several times larger. Anyone reading just the merge diff misses the migration sub-PRs. 

## Open issues / follow-up

- **Migration tool for new columns** — needed before any production DB can upgrade. Sub-PR #6856 ("Refactor authorization system catalog") and #6857 ("Add timestamps and invalidated time for additional system catalog") may carry the missing migration; worth ingesting next if the upgrade path is wanted.
- **Server-side consumer of `compile_response::dependencies`** — at baseline `175442fc` the deps vector is unpacked but no code reads it. Likely consumed by a future `_db_dependencies` table or schema-DDL-invalidation mechanism. Future ingest target.
- **`sql_data_access` parser plumbing** — the column is reserved, the enum is defined, the catalog write site is wired, but no parser path produces a value other than `UNKNOWN`. Parser/grammar work needed.

## Baseline impact

- Before: `175442fc858bd0075165729756745be6f8928036`
- After: `175442fc858bd0075165729756745be6f8928036` (unchanged — case b)
- Bump triggered: `false`
- Logged: see [[log]] entry `[2026-04-26] pr-ingest PR #6443`.

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/6443
- Jira: [CBRD-25862](https://jira.cubrid.org/browse/CBRD-25862), [CBRD-26036](https://jira.cubrid.org/browse/CBRD-26036), [CBRD-25974](https://jira.cubrid.org/browse/CBRD-25974) (db_authorizations removal)
- Components touched: [[components/system-catalog]], [[components/object]], [[components/schema-manager]], [[components/storage]], [[components/authenticate]], [[components/sp]], [[components/execute-statement]], [[components/execute-schema]], [[components/transaction]], [[components/server-boot]], [[components/utility-binaries]], [[components/communication]], [[components/btree]], [[components/db-value]]
- Sources: [[sources/cubrid-src-storage]], [[sources/cubrid-src-object]], [[sources/cubrid-src-sp]], [[sources/cubrid-src-compat]]
- Adjacent PRs: future #6856 / #6857 (likely contain the missing migration); previous #6753 ([[prs/PR-6753-optimizer-histogram-support]]) bumped `CNT_CATCLS_OBJECTS` from 6 to 8 — this PR did not touch it.
