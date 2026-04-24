---
type: component
parent_module: "[[components/object|object]]"
path: "src/object/"
status: active
purpose: "System catalog tables (_db_*) and information-schema virtual views — bootstrap install and SQL query definitions"
key_files:
  - "schema_system_catalog_install.cpp (bootstrap: create catalog tables + info_schema views)"
  - "schema_system_catalog_constants.h (CT_* / CTV_* name constants)"
  - "schema_system_catalog_install_query_spec.cpp (SQL bodies for all info_schema views)"
  - "schema_system_catalog.hpp / schema_system_catalog.cpp (helpers for catalog name comparison)"
  - "schema_system_catalog_builder.hpp (DSL for adding catalog class columns)"
  - "schema_system_catalog_definition.hpp (column-definition helpers)"
public_api:
  - "catcls_add_data_type(class_mop) — insert rows into _db_data_type at bootstrap"
  - "sm_define_view_class_spec() → const char* — SQL string for db_class view"
  - "sm_define_view_attribute_spec() → const char* — SQL for db_attribute view"
  - "CT_CLASS_NAME / CT_ATTRIBUTE_NAME / etc. — string constants for catalog table names"
  - "CTV_CLASS_NAME / CTV_ATTRIBUTE_NAME / etc. — string constants for info_schema view names"
tags:
  - component
  - cubrid
  - catalog
  - information-schema
  - client
related:
  - "[[components/object|object]]"
  - "[[components/schema-manager|schema-manager]]"
  - "[[components/authenticate|authenticate]]"
  - "[[Error Handling Convention]]"
  - "[[components/parser|parser]]"
created: 2026-04-23
updated: 2026-04-23
---

# System Catalog + Information Schema (`src/object/`)

CUBRID's system catalog is a set of physical `_db_*` tables created at database initialization. The information schema is a layer of virtual views (`db_*`) defined as SQL `SELECT` queries over those physical tables. Both are installed by `schema_system_catalog_install.cpp`.

> [!key-insight] Two-layer design
> Physical tables (`_db_class`, `_db_attribute`, …) store raw catalog data as ordinary heap rows. Information-schema views (`db_class`, `db_attribute`, …) wrap them with privilege filtering (using `AUTH_CHECK_CLASS`, `CURRENT_USER_GROUPS_SUBQUERY`, etc.) and friendly column names. Schema manager code touches the physical tables; user SQL touches only the views.

## Physical Catalog Tables

Defined by constants in `schema_system_catalog_constants.h`:

| Constant | Table | Content |
|----------|-------|---------|
| `CT_CLASS_NAME` | `_db_class` | All tables, views, system tables |
| `CT_ATTRIBUTE_NAME` | `_db_attribute` | Column definitions (all representations) |
| `CT_DOMAIN_NAME` | `_db_domain` | Data-type domain info (set element types, etc.) |
| `CT_INDEX_NAME` | `_db_index` | Index metadata |
| `CT_INDEXKEY_NAME` | `_db_index_key` | Per-column index key info |
| `CT_CLASSAUTH_NAME` | `_db_auth` | Authorization grants |
| `CT_SERIAL_NAME` | `_db_serial` | AUTO_INCREMENT sequences |
| `CT_STORED_PROC_NAME` | `_db_stored_procedure` | SP metadata |
| `CT_STORED_PROC_ARGS_NAME` | `_db_stored_procedure_args` | SP argument metadata |
| `CT_STORED_PROC_CODE_NAME` | `_db_stored_procedure_code` | SP source/bytecode |
| `CT_PARTITION_NAME` | `_db_partition` | Partition metadata |
| `CT_TRIGGER_NAME` | `_db_trigger` | Trigger definitions |
| `CT_COLLATION_NAME` | `_db_collation` | Collation info |
| `CT_CHARSET_NAME` | `_db_charset` | Character set info |
| `CT_SERVER_NAME` | `_db_server` | Linked server (dblink) info |
| `CT_SYNONYM_NAME` | `_db_synonym` | Synonym definitions |
| `CT_HA_APPLY_INFO_NAME` | `_db_ha_apply_info` | HA replication state |
| `CT_DUAL_NAME` | `dual` | Single-row dummy table |
| `AU_USER_CLASS_NAME` (`CT_USER_NAME`) | `db_user` | Database users (also the auth user class) |

## Information-Schema Views

| Constant | View | Content |
|----------|------|---------|
| `CTV_CLASS_NAME` | `db_class` | Tables and views visible to current user |
| `CTV_ATTRIBUTE_NAME` | `db_attribute` | Columns visible to current user |
| `CTV_INDEX_NAME` | `db_index` | Indexes visible to current user |
| `CTV_AUTH_NAME` | `db_auth` | Grants visible to current user |
| `CTV_STORED_PROC_NAME` | `db_stored_procedure` | SPs |
| `CTV_SERIAL_NAME` | `db_serial` | AUTO_INCREMENT serials |
| `CTV_PARTITION_NAME` | `db_partition` | Partition info |
| `CTV_TRIGGER_NAME` | `db_trigger` | Triggers |
| `CTV_VCLASS_NAME` | `db_vclass` | Views only |
| `CTV_SUPER_CLASS_NAME` | `db_direct_super_class` | Inheritance hierarchy |
| `CTV_SYNONYM_NAME` | `db_synonym` | Synonyms |
| `CTV_SERVER_NAME` | `db_server` | Linked servers |
| `CTV_COLLATION_NAME` | `db_collation` | Collations |
| `CTV_CHARSET_NAME` | `db_charset` | Character sets |

## `CNT_CATCLS_OBJECTS` Invariant

A compile-time constant (`constexpr int CNT_CATCLS_OBJECTS` — 6 at baseline, living inside `schema_class_truncator.cpp`) counts how many catalog classes contain `DB_TYPE_OBJECT` columns that reference other catalog classes. `schema_class_truncator.cpp` uses it to gate truncate-time catalog integrity checks (`cnt_refers = CNT_CATCLS_OBJECTS + 1` drives a `SELECT` against `_db_domain`). Any PR that adds a new catalog class with an OBJECT-typed column must bump this counter in lockstep — a paired QA test asserts the value. The constant is a candidate for relocation into `schema_system_catalog_constants.h` so it lives next to the other catalog-surface constants.

## Bootstrap Install Flow

At database creation, `schema_system_catalog_install.cpp` is called to:
1. Create all physical `_db_*` tables using the builder DSL (`schema_system_catalog_builder.hpp`).
2. Insert initial rows (e.g. `catcls_add_data_type` populates `_db_data_type` with all `DB_TYPE_*` names).
3. Define all `db_*` virtual views by executing `CREATE VIEW ... AS <spec>` where the spec strings come from `schema_system_catalog_install_query_spec.cpp`.

The builder DSL uses lambdas to build `DB_VALUE` column values:
```cpp
static std::function<int (DB_VALUE *)> make_int_value_fn (int num) { ... }
static std::function<int (DB_VALUE *)> make_double_value_fn (double num) { ... }
static std::function<int (DB_VALUE *)> make_numeric_value_fn (const char *str) { ... }
```

## Query Spec Files and Formatting Rules

`schema_system_catalog_install_query_spec.cpp` contains functions like `sm_define_view_class_spec()` that return static `char[]` strings holding `SELECT` statements.

> [!warning] CI-enforced formatting — 9 strict rules
> Any violation fails CI (`check.yml`). The rules (from AGENTS.md and the file's own comment block):
> 1. Indent: 1 tab + 2 spaces for statements; 2 extra spaces for `CASE` body.
> 2. Lines do not start with a space.
> 3. All lines end with a space — EXCEPT before `)` or after `(`.
> 4. Space before `(` and after `)`, `{`, `}`.
> 5. Operators `+` and `=` surrounded by spaces.
> 6. Line breaks after `SELECT`, `FROM`, `WHERE`, `ORDER BY` and before `AND`, `OR`.
> 7. `WHEN` and `THEN` on one line if total < 120 chars.
> 8. Always use `AS` for aliases.
> 9. Comment CAST data-type changes: `CAST (x AS VARCHAR(255)) /* string -> varchar(255) */`.

Additionally, auth macros must be used for access control:
- `AUTH_CHECK_CLASS()` — visibility filter for tables accessible to current user.
- `AUTH_CHECK_OWNER()` — owner-only visibility.
- `AUTH_CHECK_DBA` — DBA-only rows.
- `CURRENT_USER_GROUPS_SUBQUERY` — subquery that resolves group membership.

## SERIAL Ranges

A notable constant defined in the install code:
- `MINVALUE` = −10^36
- `MAXVALUE` = 10^37
- These are intentionally **not** ±10^38 (numeric precision boundary).

## Where to Make Changes

| Task | File |
|------|------|
| Add a new catalog table/column | `schema_system_catalog_install.cpp` + `schema_system_catalog_constants.h` |
| Add a new info-schema view | `schema_system_catalog_install.cpp` (create view call) + `*_install_query_spec.cpp` (SQL spec) |
| Reference a catalog table by name | Always use the `CT_*` / `CTV_*` constant, never a raw string |
| Change a view's privilege logic | Use/extend `AUTH_CHECK_CLASS()` macro in the query spec |

## Related

- Parent: [[components/object|object]]
- [[components/schema-manager|schema-manager]] — DDL that writes to these catalog tables
- [[components/authenticate|authenticate]] — `AUTH_CHECK_*` macros expand to auth system queries
- [[components/parser|parser]] — info-schema views are parsed as ordinary SQL
- [[Error Handling Convention]] — 6-place rule applies when adding new error codes in install functions
