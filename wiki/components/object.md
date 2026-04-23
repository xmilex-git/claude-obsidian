---
type: component
parent_module: "[[modules/src|src]]"
path: "src/object/"
status: active
purpose: "Client-side schema, authorization, system catalog, object/class representation, LOB locator, workspace, triggers"
key_files:
  - "schema_manager.c / schema_manager.h (sm_ — class/table lifecycle, constraint management)"
  - "schema_template.c / schema_template.h (smt_ — transactional schema edit templates)"
  - "class_object.c / class_object.h (SM_CLASS, SM_ATTRIBUTE, SM_CONSTRAINT — in-memory class repr)"
  - "authenticate.c / authenticate.h / authenticate_context.hpp (au_ — users, privileges)"
  - "schema_system_catalog_install.cpp (system catalog table creation at bootstrap)"
  - "schema_system_catalog_constants.h (CT_CLASS_NAME, CT_ATTRIBUTE_NAME, etc.)"
  - "schema_system_catalog_install_query_spec.cpp (information schema SQL query bodies)"
  - "lob_locator.cpp / lob_locator.hpp (LOB locator state machine)"
  - "trigger_manager.c / trigger_manager.h (tr_ — trigger creation and execution)"
  - "work_space.c / work_space.h (ws_ — client-side object cache / workspace)"
  - "object_accessor.c / object_accessor.h (attribute get/set on MOP instances)"
  - "object_primitive.c / object_primitive.h (primitive type compare, copy, size)"
  - "schema_class_truncator.cpp (TRUNCATE TABLE implementation)"
  - "transform.c / transform_cl.c (disk <-> memory object serialization)"
  - "quick_fit.c / quick_fit.h (workspace memory allocator)"
public_api:
  - "sm_update_class(template_, classmop) — commit a schema change"
  - "smt_edit_class_mop(class_, auth) — open a schema edit template"
  - "sm_add_constraint / sm_drop_constraint / sm_drop_index"
  - "sm_find_class(name) — look up a class MOP by name"
  - "sm_delete_class_mop(op, cascade) — drop a table/view"
  - "sm_truncate_class(mop, cascade)"
  - "au_ctx() — get/create authenticate_context singleton"
  - "AU_DISABLE(save) / AU_ENABLE(save) — bypass privilege checks"
  - "lob_locator_find / lob_locator_add / lob_locator_change_state / lob_locator_drop"
  - "catcls_add_data_type(class_mop) — bootstrap data-type catalog rows"
tags:
  - component
  - cubrid
  - object
  - schema
  - auth
  - client
related:
  - "[[modules/src|src]]"
  - "[[components/storage|storage]]"
  - "[[components/external-storage|external-storage]]"
  - "[[components/transaction|transaction]]"
  - "[[components/parser|parser]]"
  - "[[components/schema-manager|schema-manager]]"
  - "[[components/system-catalog|system-catalog]]"
  - "[[components/authenticate|authenticate]]"
  - "[[components/lob-locator|lob-locator]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Error Handling Convention]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/object/` — Object / Schema / Auth / Info-Schema Layer

`src/object/` is the client-side (non-`SERVER_MODE`) layer that owns:
- **Schema management** — DDL for tables, views, indexes, foreign keys, partitions, serials, truncate.
- **System catalog** — the physical `_db_*` tables and information-schema virtual views, installed at database creation time.
- **Authorization** — users, groups, role inheritance, privilege caching, password hashing.
- **In-memory class representation** — `SM_CLASS`, `SM_ATTRIBUTE`, `SM_CONSTRAINT` structures used by schema manager, executor, and type checker.
- **LOB locator** — transaction-aware state machine tracking BLOB/CLOB locator strings.
- **Workspace (object cache)** — client-side `MOP` cache and memory allocator.
- **Triggers** — schema-level trigger creation and event routing.

> [!warning] Client-side only
> All files in this directory compile with `#error Does not belong to server module` under `SERVER_MODE`. Schema operations execute on the client (or SA_MODE in-process), never directly in `cub_server`.

## Architecture Overview

```
DDL SQL text
  → Parser (src/parser/) → PT_NODE
  → Schema Manager (schema_manager.c)
      → SM_TEMPLATE (schema_template.c)  — transactional schema edit buffer
      → sm_update_class()                — commit to workspace
          → transform.c                  — disk ↔ memory serialization
          → work_space.c                 — MOP cache update
          → System Catalog tables        — _db_class, _db_attribute, _db_index, …
          → authenticate.c               — privilege check before commit

Workspace (work_space.c)
  ← class descriptors (SM_CLASS) cached per-session
  ← object instances fetched by MOP from heap (src/storage/heap_file.c)

Auth context (authenticate_context.hpp)
  → _db_user, _db_auth catalog tables
  → AU_DISABLE / AU_ENABLE macros for internal bypass

LOB locator (lob_locator.cpp)
  → in CS_MODE: network call → server transaction_transient
  → in SA_MODE: xtx_find_lob_locator() directly
  ↔ physical storage: src/storage/es.c (POSIX / OWFS backend)
```

## Sub-Systems

| Sub-system | File(s) | Wiki page |
|------------|---------|-----------|
| Schema manager | `schema_manager.c/h`, `schema_template.c/h` | [[components/schema-manager]] |
| System catalog + info_schema | `schema_system_catalog_install.cpp`, `*_constants.h`, `*_install_query_spec.cpp` | [[components/system-catalog]] |
| Authorization | `authenticate.c/h`, `authenticate_context.hpp`, `authenticate_cache.hpp` | [[components/authenticate]] |
| Class representation | `class_object.c/h` | (inline below) |
| LOB locator | `lob_locator.cpp/hpp` | [[components/lob-locator]] |
| Triggers | `trigger_manager.c/h` | (not yet a dedicated page) |
| Workspace | `work_space.c/h`, `quick_fit.c/h` | (inline below) |
| Object accessor | `object_accessor.c/h` | (inline below) |
| Transform | `transform.c`, `transform_cl.c` | (inline below) |

## Class Representation (`class_object.h`)

The `SM_CLASS` structure is the in-memory descriptor for every table, view, and system table:

| Struct | Role |
|--------|------|
| `SM_CLASS_HEADER` | Common prefix: object header, meta-type tag, name, `ch_rep_dir` (OID to representation directory), `ch_heap` (`HFID`) |
| `SM_CLASS` | Full class: attribute list, method list, constraint list, super-class list, property list, flags |
| `SM_ATTRIBUTE` | Column: name, domain (`TP_DOMAIN`), default value, constraint cache, flags, order, `auto_increment` MOP |
| `SM_CONSTRAINT` | Per-attribute constraint cache: type, `BTID`, has_function flag |
| `SM_CLASS_CONSTRAINT` | Full constraint entry (linked to class, not just attribute) |
| `SM_FOREIGN_KEY_INFO` | FK detail: referenced class/attrs, OID, BTID, delete/update action |

Key type tags:
- `SM_CLASS_CT` — normal base table
- `SM_VCLASS_CT` — view (virtual class)
- `SM_ADT_CT` — abstract data type pseudo-class

Class flags (`SM_CLASS_FLAG`): `SM_CLASSFLAG_SYSTEM`, `SM_CLASSFLAG_WITHCHECKOPTION`, `SM_CLASSFLAG_REUSE_OID`, `SM_CLASSFLAG_SUPPLEMENTAL_LOG`.

## Schema Edit Template Pattern

DDL changes follow a strict three-step template pattern:

```
smt_edit_class_mop(mop, auth)    → SM_TEMPLATE* (editable copy)
  smt_add_attribute(...)         — mutate the template
  smt_add_constraint(...)
  ...
sm_update_class(template_, mop) → commit changes to workspace + catalog
```

`sm_finish_class` is a lower-level variant. `sm_update_class_with_auth` adds privilege checking inline.

> [!key-insight] Template is a copy-on-write buffer
> `SM_TEMPLATE` is a mutable working copy of an `SM_CLASS`. Calling `sm_update_class` atomically replaces the live class descriptor and writes the changes to the system catalog, all within the current transaction. Rolling back the transaction discards the template changes.

## Workspace (`work_space.c`)

The workspace is the client-side object cache. It maps `OID → MOP` and maintains:
- Dirty-object tracking (objects modified but not yet flushed to server)
- Memory via `quick_fit.c` — a pool allocator for small fixed-size objects
- Prefix `ws_` for all functions

## Function-Prefix Summary

| Prefix | Module | Example |
|--------|--------|---------|
| `sm_` | schema_manager.c | `sm_update_class`, `sm_find_class` |
| `smt_` | schema_template.c | `smt_edit_class_mop`, `smt_add_attribute` |
| `au_` | authenticate.c | `au_check_authorization`, `au_set_user` |
| `tr_` | trigger_manager.c | `tr_create_trigger`, `tr_execute_event` |
| `ws_` | work_space.c | `ws_mop`, `ws_flush_all` |
| `lob_locator_` | lob_locator.cpp | `lob_locator_find`, `lob_locator_add` |
| `catcls_` | schema_system_catalog_install.cpp | `catcls_add_data_type` |

## Cross-Cutting Concerns

### LOB: Locator + Physical Storage

LOB handling is split across two directories:
- **This directory**: `lob_locator.cpp` — transaction-level state machine (`LOB_TRANSIENT_CREATED → LOB_PERMANENT_CREATED`, etc.).
- **`src/storage/es.c`**: physical byte storage on POSIX / OWFS filesystem.

See [[components/lob-locator]] and [[components/external-storage]].

### Info-Schema: Strict Formatting Rules

Query specs in `schema_system_catalog_install_query_spec.cpp` are CI-enforced with 9 formatting rules (tab + 2-space indent, mandatory trailing space, specific line-break positions). Violations fail CI. See [[components/system-catalog]].

### Schema Changes are Transactional

All `sm_update_class` calls run inside a client-side transaction. Rollback reverts the `SM_CLASS` descriptor and catalog rows. This is coordinated with [[components/transaction]] via the workspace dirty-object mechanism.

## Common Bug Locations

| Symptom | File | Entry point |
|---------|------|-------------|
| Wrong column default after ALTER TABLE | `class_object.h` | `SM_ATTRIBUTE.original_value` vs `value` semantics |
| Auth check skipped | `authenticate.c` | `AU_DISABLE` macro left enabled |
| Info-schema view returns wrong data | `*_install_query_spec.cpp` | SQL in `sm_define_view_*_spec()` |
| Catalog row missing after CREATE TABLE | `schema_system_catalog_install.cpp` | `catcls_*` install functions |
| LOB file leaked after rollback | `lob_locator.cpp` | `LOB_TRANSIENT_CREATED → LOB_UNKNOWN` state path |
| Constraint not dropped on DROP TABLE | `schema_manager.c` | `sm_delete_class_mop` cascade logic |

## Related

- Parent: [[modules/src|src]]
- [[components/schema-manager|schema-manager]] — DDL lifecycle detail
- [[components/system-catalog|system-catalog]] — catalog tables and info_schema views
- [[components/authenticate|authenticate]] — users, groups, privilege caching
- [[components/lob-locator|lob-locator]] — LOB transaction state machine
- [[components/storage|storage]] — heap and B-tree that back the catalog tables
- [[components/external-storage|external-storage]] — LOB physical byte storage
- [[components/parser|parser]] — DDL text → PT_NODE (consumed by schema manager)
- [[components/transaction|transaction]] — schema changes participate in MVCC transactions
- [[Error Handling Convention]] — 6-place error code rule applies here too
- Source: [[sources/cubrid-src-object]]
