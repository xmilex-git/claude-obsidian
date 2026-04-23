---
type: source
title: "CUBRID src/object/ — Schema, Auth, Catalog, LOB"
date: 2026-04-23
tags:
  - source
  - cubrid
  - codebase
  - schema
  - auth
  - catalog
  - lob
status: ingested
related:
  - "[[components/object|object]]"
  - "[[components/schema-manager|schema-manager]]"
  - "[[components/system-catalog|system-catalog]]"
  - "[[components/authenticate|authenticate]]"
  - "[[components/lob-locator|lob-locator]]"
---

# CUBRID `src/object/` — Source Summary

Ingested: 2026-04-23. Direct read of source files and `src/object/AGENTS.md`.

## What Was Read

| File | Purpose |
|------|---------|
| `AGENTS.md` | Directory overview, key-file table, formatting rules, system catalog table list, conventions |
| `schema_manager.h` | Full public API: `sm_update_class`, `sm_delete_class_mop`, constraint management, class lookup, statistics |
| `schema_template.h` | Template DSL: `smt_def_class`, `smt_edit_class_mop`, `smt_add_attribute*`, `smt_quit` |
| `class_object.h` | `SM_CLASS_HEADER`, `SM_CLASS`, `SM_ATTRIBUTE`, `SM_CONSTRAINT`, `SM_CLASS_TYPE`, `SM_CLASS_FLAG` enums |
| `authenticate.h` | Legacy macro shim over `authenticate_context.hpp`; `AU_DISABLE/ENABLE` pattern |
| `authenticate_context.hpp` | `authenticate_context` C++ class: user stack, password buffers, `push_user/pop_user`, `login`, `start`, `install` |
| `schema_system_catalog_constants.h` | All `CT_*` and `CTV_*` string constants for catalog table and view names |
| `schema_system_catalog_install.cpp` | Bootstrap entry points: `catcls_add_data_type`; value-builder lambdas |
| `schema_system_catalog_install_query_spec.cpp` | `sm_define_view_*_spec()` functions; 9-rule formatting comment block |
| `lob_locator.hpp` | `LOB_LOCATOR_STATE` enum; 7-function API |
| `lob_locator.cpp` | State machine; `CS_MODE` network dispatch vs `SA_MODE` direct call |

## Key Facts Extracted

1. **Template pattern is the only DDL path**: every schema change must go through `smt_edit_class_mop → mutations → sm_update_class`. Bypassing this corrupts the `SM_CLASS` in-memory descriptor.

2. **`authenticate_context` is now the real implementation**: the old global `Au_*` variables are `#define`d macros that forward to `au_ctx()`. Code that directly touches the old globals is being migrated.

3. **Info-schema view specs have 9 CI-enforced formatting rules**: violations fail CI. The rules are documented both in `AGENTS.md` and at the top of `schema_system_catalog_install_query_spec.cpp`.

4. **SERIAL bounds are ±10^36..10^37, not ±10^38**: this is explicit and intentional (numeric precision boundary avoidance).

5. **LOB locator is a thin client wrapper**: the actual state registry lives in `transaction_transient.hpp` server-side. `lob_locator.cpp` dispatches via network (CS_MODE) or direct call (SA_MODE).

6. **`SM_ATTRIBUTE.original_value` vs `value`**: `original_value` is the first-ever default value for a column, preserved across subsequent `ALTER TABLE ... CHANGE DEFAULT` operations. It is used to backfill old representations when scanning pre-`ALTER` rows.

7. **Class header embeds `ch_heap` (HFID) and `ch_rep_dir` (OID)**: every `SM_CLASS_HEADER` carries a direct reference to its heap file and representation directory — no indirection needed during scan.

## Pages Created

- [[components/object|object]] — hub page
- [[components/schema-manager|schema-manager]] — DDL lifecycle, template pattern, constraint management
- [[components/system-catalog|system-catalog]] — catalog tables, info-schema views, formatting rules
- [[components/authenticate|authenticate]] — users, groups, privilege caching, execution-rights stack
- [[components/lob-locator|lob-locator]] — LOB state machine, CS/SA dispatch

## Follow-Up Items

- [ ] Trigger manager (`trigger_manager.c`) — not yet a dedicated wiki page.
- [ ] Workspace (`work_space.c`) — deserves its own page; MOP cache invalidation is a common bug area.
- [ ] `transform.c` / `transform_cl.c` — disk ↔ memory object serialization; needed for understanding representation versioning.
- [ ] Flow page: `sm_update_class` lifecycle from DDL parse to catalog commit.
- [ ] Flow page: LOB write path (client `lob_locator_add` → server `es_create_file` → commit cleanup).
