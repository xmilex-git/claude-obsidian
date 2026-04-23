---
type: component
title: "execute-schema — DDL Execution Helpers"
parent_module: "[[modules/src|src]]"
path: src/query/execute_schema.{c,h}
status: developing
key_files:
  - src/query/execute_schema.c
  - src/query/execute_schema.h
public_api:
  - do_create_entity
  - do_create_local
  - do_add_attributes
  - do_add_constraints
  - do_add_queries
  - do_check_rows_for_null
  - do_drop_partitioned_class
  - do_is_partitioned_subclass
  - do_get_partition_parent
  - do_rename_partition
  - do_recreate_func_index_constr
  - do_recreate_filter_index_constr
  - init_update_data
tags:
  - cubrid
  - ddl
  - schema
  - partition
  - alter-table
  - client-side
---

# execute-schema — DDL Execution Helpers

> [!key-insight]
> `execute_schema.c` (~16 K lines) is **client-side** (`#if defined(SERVER_MODE) #error`) and handles the actual execution of DDL statements that mutate the schema catalog. It uses the `SM_TEMPLATE` (schema manager template) edit-commit pattern exclusively: build a template, fill in attribute/constraint changes, then commit via `smt_finish_class()`.

## Purpose

`execute_schema.c` provides the DDL execution bodies called by `execute_statement.c`. Key responsibilities:

- **CREATE TABLE / VIEW / CLASS**: `do_create_local` / `do_create_entity` — build a `DB_CTMPL`, populate attributes, methods, constraints, then commit.
- **ALTER TABLE**: attribute change analysis (`SM_ATTR_CHG_SOL` enum), column type compatibility matrix, constraint add/drop/rebuild.
- **ALTER INDEX / CREATE INDEX / DROP INDEX**: issue calls into `sm_add_index` / `sm_drop_index`; re-create functional/filter index constraints on column rename.
- **Partition management**: `do_drop_partitioned_class`, `do_rename_partition`, partition sub-class checks.
- **Client-side UPDATE data prep**: `init_update_data` — allocates `CLIENT_UPDATE_INFO` arrays used by the row-by-row update path.
- **User/grant/revoke**: `do_create_user`, `do_drop_user`, `do_alter_user`, `do_grant`, `do_revoke`.
- **TRUNCATE**: uses savepoint `tRUnCATE` + `do_drop_partitioned_class` for partition truncation.

---

## Public Entry Points

| Signature | Role |
|-----------|------|
| `int do_create_entity(PARSER_CONTEXT*, PT_NODE*)` | Top-level CREATE TABLE/VIEW/CLASS dispatcher |
| `int do_create_local(PARSER_CONTEXT*, DB_CTMPL*, PT_NODE*, DB_QUERY_TYPE*)` | Core create: populates template, commits |
| `int do_add_attributes(PARSER_CONTEXT*, DB_CTMPL*, PT_NODE* atts, PT_NODE* constraints, …)` | Add column definitions to template |
| `int do_add_constraints(DB_CTMPL*, PT_NODE*)` | Add constraint definitions to template |
| `int do_check_fk_constraints(DB_CTMPL*, PT_NODE*)` | Validate FK constraint references |
| `int do_add_methods(…)` / `do_add_method_files(…)` | OO-era method additions |
| `int do_add_supers(…)` / `do_add_resolutions(…)` | Inheritance chain setup |
| `int do_check_rows_for_null(MOP class_mop, const char* att_name, bool* has_nulls)` | Scans heap to verify NOT NULL compliance before constraint add |
| `int do_drop_partitioned_class(MOP, int drop_sub_flag, bool cascade)` | Drop all partition sub-classes |
| `int do_is_partitioned_subclass(int*, const char* classname, char* keyattr)` | Check if class is a partition |
| `int do_get_partition_parent(DB_OBJECT*, MOP* parentop)` | Resolve partition → parent class |
| `int do_rename_partition(MOP old_class, const char* newname)` | Rename partition sub-class |
| `int do_check_partitioned_class(DB_OBJECT*, int check_map, char* keyattr)` | Assert partition structure |
| `int do_drop_partition_list(MOP, PT_NODE* name_list, DB_CTMPL*)` | Drop listed partitions |
| `int do_recreate_func_index_constr(…)` | Recreate functional index expression after column rename |
| `int do_recreate_filter_index_constr(…)` | Recreate filter predicate index after column rename |
| `int init_update_data(PARSER_CONTEXT*, PT_NODE*, CLIENT_UPDATE_INFO**, int*, CLIENT_UPDATE_CLASS_INFO**, int*, DB_VALUE**, int*, bool)` | Set up per-column update assignment descriptors |

---

## ALTER TABLE Attribute-Change Decision Matrix

ALTER TABLE column changes go through a `SM_ATTR_CHG_SOL` assessment:

```
SM_ATTR_CHG_NOT_NEEDED (0)   — only metadata (comment, rename)
SM_ATTR_CHG_ONLY_SCHEMA (1)  — schema-only change (widen type, compatible)
SM_ATTR_CHG_WITH_ROW_UPDATE (2)  — requires full table rewrite
SM_ATTR_CHG_BEST_EFFORT (3)  — row rewrite; likely to fail at row scan time
```

The `ATT_CHG_*` bitmask flags per attribute property drive this decision:

```
ATT_CHG_PROPERTY_PRESENT_OLD / _NEW — old/new presence
ATT_CHG_TYPE_NOT_SUPPORTED          — type conversion not possible
ATT_CHG_TYPE_NEED_ROW_CHECK         — must scan rows to verify
ATT_CHG_TYPE_SET_CLS_COMPAT         — object domain subclass compat
ATT_CHG_TYPE_PSEUDO_UPGRADE         — collation change: index rebuild needed
```

> [!key-insight]
> When `SM_ATTR_CHG_WITH_ROW_UPDATE` is required, CUBRID performs an internal `SELECT * FROM t` + `INSERT INTO t_tmp` cycle (not an in-place rewrite). This means a full table copy is materialized. The change is wrapped in a named savepoint (`cHANGEaTTR`) for rollback.

---

## Catalog Mutation Sequencing

DDL uses the `SM_TEMPLATE` edit-commit pattern from [[components/schema-manager]]:

```
smt_edit_class(class_mop)          → SM_TEMPLATE* tmpl
do_add_attributes(…, tmpl, …)      → smt_add_attribute / smt_add_constraint
do_add_constraints(…, tmpl, …)     → smt_add_constraint_with_name
do_check_rows_for_null(…)          → heap scan (if NOT NULL being added)
smt_finish_class(tmpl, &class_mop) → flushes catalog; validates
locator_flush_class(class_mop)     → writes _db_class catalog heap page
```

Savepoint names are constant strings (e.g., `"aDDaTTRmTHD"`, `"cREATEeNTITY"`, `"aLTERiNDEX"`) to ensure uniqueness. Partition-related savepoints use `"pARTITION*"` prefix.

---

## Partition Handling

The `_db_partition` catalog class stores partition metadata. Constants:
- `PARTITION_CATALOG_CLASS = "_db_partition"`
- `CLASS_ATT_NAME = "class_name"`, `CLASS_IS_PARTITION = "partition_of"`

Partition operations:
```
do_drop_partitioned_class
    for each sub-class in partition list:
        if drop_sub_flag & IS_CASCADE: cascade constraints
        db_drop_class(sub_class_mop)
    db_drop_class(parent_mop)

do_rename_partition(old_class, newname)
    sm_rename_class(old_class, newname)   ← schema manager rename
```

> [!warning]
> Partition renames and drops require `AU_DISABLE` (auth bypass) because partition sub-classes are internal system objects not directly accessible to users. Forgetting `AU_ENABLE` after `AU_DISABLE` will leave the session in an auth-disabled state.

---

## CLIENT_UPDATE_INFO — Row-by-Row Update Structures

`execute_schema.h` exports two structs used by the client-side row update path:

```c
CLIENT_UPDATE_CLASS_INFO {
    PT_NODE *spec;         // PT_SPEC for this class
    DB_VALUE *oid;         // OID of last updated tuple
    PT_NODE *check_where;  // CHECK OPTION expr (for updatable views)
    SM_CLASS *smclass;
    DB_OBJECT *class_mop;
    int pruning_type;      // partitioned? DB_NOT_PARTITIONED / DB_PARTITION_TYPE
    CLIENT_UPDATE_INFO *first_assign;
}

CLIENT_UPDATE_INFO {
    PT_NODE *upd_col_name;      // PT_NAME of column
    DB_VALUE *db_val;           // value to assign
    bool is_const;
    DB_ATTDESC *attr_desc;
    CLIENT_UPDATE_CLASS_INFO *cls_info;
    CLIENT_UPDATE_INFO *next;
}
```

`init_update_data()` allocates these arrays from `db_private_alloc(parser->private_memory, …)`.

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Build mode | `#if defined(SERVER_MODE) #error` — client-only |
| Auth | Many functions call `AU_DISABLE` / `AU_ENABLE` to bypass ownership checks for internal catalog objects |
| Transaction | Each DDL operation has its own savepoint; rollback restores schema on failure |
| Memory | `DB_CTMPL` lifetime ends at `smt_finish_class` or `smt_quit`; attributes in parser arena |
| Reentrancy | Not reentrant; intended for single-threaded CAS per connection |

---

## Lifecycle

```
Per DDL statement:
    do_execute_statement calls do_alter / do_create_entity / etc.
    └─ SM_TEMPLATE allocated by smt_edit_class or smt_def_class
    └─ attributes/constraints added incrementally
    └─ do_check_rows_for_null: heap scan before adding NOT NULL
    └─ smt_finish_class: atomic catalog update + schema cache invalidation
    └─ locator_flush_class: write to heap
    └─ XASL cache decache for affected class (via xasl_cache_remove_class)
```

---

## Related

- [[components/execute-statement]] — calls all `do_*` functions here from the statement dispatcher
- [[components/schema-manager]] — `SM_TEMPLATE`, `smt_*` API, constraint management
- [[components/system-catalog]] — `_db_class`, `_db_attribute`, `_db_partition` tables mutated here
- [[components/partition-pruning]] — runtime partition elimination using metadata written here
- [[components/authenticate]] — `AU_DISABLE`/`AU_ENABLE` used throughout
- [[flows/ddl-execution-path]]
- [[Memory Management Conventions]]
