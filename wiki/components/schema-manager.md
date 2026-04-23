---
type: component
parent_module: "[[components/object|object]]"
path: "src/object/"
status: active
purpose: "Class/table schema lifecycle: CREATE, ALTER, DROP, TRUNCATE, constraint management, index management, class lookup"
key_files:
  - "schema_manager.c / schema_manager.h (sm_ prefix — primary entry points)"
  - "schema_template.c / schema_template.h (smt_ prefix — edit template pattern)"
  - "schema_class_truncator.cpp (TRUNCATE TABLE logic)"
  - "class_object.h (SM_CLASS, SM_ATTRIBUTE, SM_CONSTRAINT data structures)"
public_api:
  - "sm_update_class(template_, classmop) — commit a schema change"
  - "sm_update_class_with_auth(template_, classmop, auth, lock_hierarchy)"
  - "sm_finish_class(template_, classmop) — low-level commit"
  - "sm_delete_class_mop(op, is_cascade_constraints)"
  - "sm_add_constraint(classop, type, name, att_names, ...)"
  - "sm_drop_constraint(classop, type, name, att_names, ...)"
  - "sm_drop_index(classop, constraint_name)"
  - "sm_find_class(name) — MOP lookup by name"
  - "sm_find_synonym(name) — synonym resolution"
  - "sm_rename_class(op, new_name)"
  - "sm_truncate_class(class_mop, is_cascade)"
  - "sm_is_system_class(op) — test for system table"
  - "sm_get_class_type(class_) — SM_CLASS_CT / SM_VCLASS_CT / SM_ADT_CT"
  - "sm_att_info(classop, name, id, domain, shared, class_attr)"
  - "sm_update_statistics(classop, fullscan)"
  - "smt_def_class(name) — create template for new class"
  - "smt_edit_class_mop(class_, auth) — create template for existing class"
  - "smt_add_attribute_w_dflt(...)"
  - "smt_quit(template_) — discard template without commit"
tags:
  - component
  - cubrid
  - schema
  - client
related:
  - "[[components/object|object]]"
  - "[[components/system-catalog|system-catalog]]"
  - "[[components/authenticate|authenticate]]"
  - "[[components/parser|parser]]"
  - "[[components/transaction|transaction]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# Schema Manager (`src/object/schema_manager.c`)

The schema manager is the DDL engine on the client side. It translates `PT_NODE` DDL operations from the parser into mutations on `SM_CLASS` descriptors and system catalog rows, all within the current transaction.

> [!warning] Client-side only
> `schema_manager.h` contains `#error Does not belong to server module` under `SERVER_MODE`. All schema operations run on the client (or in SA_MODE in-process). The server never directly executes schema DDL.

## Template Pattern

Every schema mutation goes through an `SM_TEMPLATE` (a mutable working copy of an `SM_CLASS`):

```
Step 1: Open a template
  SM_TEMPLATE *tmpl = smt_edit_class_mop (class_mop, DB_AUTH_ALTER);
  // or for CREATE: smt_def_class ("my_table");

Step 2: Accumulate changes
  smt_add_attribute_w_dflt (tmpl, "col", "INTEGER", NULL, &defval, ...);
  smt_add_constraint (tmpl, ...);   // via schema_manager, not smt_ directly

Step 3: Commit
  sm_update_class (tmpl, &class_mop);
  // template_ pointer is now invalid

On error:
  smt_quit (tmpl);   // discard without commit
```

> [!key-insight] Copy-on-write DDL
> `smt_edit_class_mop` makes a working copy of the live `SM_CLASS`. Calling `sm_update_class` atomically replaces the descriptor and persists catalog rows within the current transaction. A rollback discards the template changes without any additional cleanup.

## Class Lifecycle Functions

| Operation | Entry point |
|-----------|-------------|
| CREATE TABLE / VIEW | `sm_finish_class` or `sm_update_class` on a `smt_def_class` template |
| ALTER TABLE (add col, etc.) | `smt_edit_class_mop` → mutations → `sm_update_class` |
| DROP TABLE | `sm_delete_class_mop(op, is_cascade_constraints)` |
| RENAME TABLE | `sm_rename_class(op, new_name)` |
| TRUNCATE TABLE | `sm_truncate_class(mop, cascade)` — two paths: `sm_truncate_using_delete` or `sm_truncate_using_destroy_heap` |
| ADD / DROP CONSTRAINT | `sm_add_constraint` / `sm_drop_constraint` |
| DROP INDEX | `sm_drop_index(classop, constraint_name)` |

## Constraint Management

`SM_CONSTRAINT_INFO` captures all information needed to drop and re-create a constraint (used during ALTER TABLE that rebuilds indexes):

```c
struct sm_constraint_info {
  char *name;
  char **att_names;
  int *asc_desc;
  int *prefix_length;
  SM_PREDICATE_INFO *filter_predicate;   // filtered index
  SM_FUNCTION_INFO *func_index_info;     // function index
  char *ref_cls_name; char **ref_attrs;  // FK
  SM_FOREIGN_KEY_ACTION fk_delete_action, fk_update_action;
  DB_CONSTRAINT_TYPE constraint_type;
  SM_INDEX_STATUS index_status;
};
```

`sm_produce_constraint_name` / `sm_produce_constraint_name_mop` / `sm_produce_constraint_name_tmpl` generate the default system-assigned constraint name.

## Class Lookup

- `sm_find_class(name)` — case-insensitive name lookup in workspace; returns `MOP`.
- `sm_find_synonym(name)` — checks `_db_synonym` first, then resolves to the target class.
- `sm_fetch_all_classes(external_list, purpose)` — returns full class list.
- `sm_fetch_all_base_classes(external_list, purpose)` — base tables only (no views).

## Statistics

- `sm_update_statistics(classop, fullscan)` — update stats for one class.
- `sm_update_all_statistics(fullscan)` — update stats for all classes.
- `sm_get_class_with_statistics(classop)` — fetch `SM_CLASS*` with stats attached.
- `sm_update_catalog_statistics(class_name, fullscan)` — stats for a system catalog table.

## TDE (Transparent Data Encryption) Integration

- `sm_set_class_tde_algorithm(classop, algo)` / `sm_get_class_tde_algorithm(classop, algo*)` — marks a class heap for TDE. Stored in `_db_class.tde_algorithm`.

## Truncate Paths

`sm_truncate_class` branches based on whether the table has reuse-OID semantics:
- `sm_truncate_using_delete` — row-by-row delete (triggers fire, MVCC visible).
- `sm_truncate_using_destroy_heap` — destroy heap file and recreate (no trigger, faster).

`schema_class_truncator.cpp` implements index-drop-and-recreate around both paths using `SM_CONSTRAINT_INFO`.

## Related

- Parent: [[components/object|object]]
- [[components/system-catalog|system-catalog]] — schema changes write to `_db_class`, `_db_attribute`, `_db_index`
- [[components/authenticate|authenticate]] — `sm_update_class_with_auth` calls `au_check_*` before commit
- [[components/parser|parser]] — DDL PT_NODE consumed here
- [[components/transaction|transaction]] — schema changes participate in client transactions
- [[Build Modes (SERVER SA CS)]] — client-only (`CS_MODE` / `SA_MODE`)
