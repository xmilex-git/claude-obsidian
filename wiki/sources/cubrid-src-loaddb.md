---
type: source
title: "CUBRID src/loaddb/ — Bulk Loader"
source_path: "src/loaddb/"
date_ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - loaddb
  - bison
  - flex
  - bulk-loader
related:
  - "[[components/loaddb|loaddb]]"
  - "[[components/loaddb-grammar|loaddb-grammar]]"
  - "[[components/loaddb-executor|loaddb-executor]]"
  - "[[components/loaddb-driver|loaddb-driver]]"
---

# Source: `src/loaddb/` — Bulk Loader

Ingested: 2026-04-23. Files read: `AGENTS.md`, `load_grammar.yy`, `load_lexer.l`, `load_driver.cpp`, `load_session.cpp`, `load_worker_manager.cpp`, `load_server_loader.cpp`, `load_db_value_converter.cpp`, `load_error_handler.cpp`, `load_common.hpp`.

## What this directory is

The `loaddb` bulk loader for CUBRID. Entirely separate from the SQL parser — it has its own LALR(1) bison grammar (`load_grammar.yy`), its own C++ flex lexer (`load_lexer.l`), its own parallel worker infrastructure, and inserts rows directly into the heap via `locator_multi_insert_force`, bypassing `XASL_NODE` execution entirely.

## Key files

| File | Role |
|------|------|
| `load_grammar.yy` | Bison LALR(1) grammar (C++ skeleton, `cubload` namespace) |
| `load_lexer.l` | Flex C++ scanner (`cubload::scanner`) |
| `load_driver.cpp` | `driver` — wires scanner + parser per batch |
| `load_session.cpp` | `session` — lifecycle, worker dispatch, ordered commit; `load_task` |
| `load_worker_manager.cpp` | Global worker pool + `worker_entry_manager` (driver pool) |
| `load_server_loader.cpp` | `server_class_installer`, `server_object_loader` — heap insertion |
| `load_db_value_converter.cpp` | String → `DB_VALUE` dispatch table (one function per LDR+DB type pair) |
| `load_error_handler.cpp` | Per-line error tracking, syntax-check mode, message catalog |
| `load_class_registry.cpp` | Session-scoped `class_registry` (class_id → class_entry + attribute metadata) |
| `load_common.hpp` | `batch` (packable), `batch_id`, `class_id` |
| `load_sa_loader.cpp` | SA_MODE entry; uses `load_object.c` legacy C layer |
| `load_object.c` | Legacy SA object construction (pre-C++ rewrite) |

## Pages created

- [[components/loaddb|loaddb]] — component hub
- [[components/loaddb-grammar|loaddb-grammar]] — grammar/lexer details
- [[components/loaddb-executor|loaddb-executor]] — server_class_installer, server_object_loader, heap insert path
- [[components/loaddb-driver|loaddb-driver]] — driver, session, worker manager

## Key insights

1. **Own grammar, not SQL**: loaddb uses `lalr1.cc` (modern C++ bison skeleton) vs. the SQL parser's legacy C skeleton. Grammar actions directly call driver callbacks — no parse tree is built.
2. **Parallel parse, serial commit**: worker threads parse batches concurrently; `session::wait_for_previous_batch()` enforces commit order via a condition variable.
3. **Direct heap bypass**: rows go `string → DB_VALUE → heap_attrinfo_transform_to_disk → locator_multi_insert_force`, skipping XASL, query executor, and the normal INSERT path entirely.
4. **HA degrades bulk insert**: when HA is enabled or errors are filtered, `locator_multi_insert_force` is replaced by per-row `locator_insert_force` + individual system operations to give accurate per-row LSAs for replication.
5. **Unit tests disabled**: the `LOADDB` module in `unit_tests/` is disabled due to compilation issues (noted in AGENTS.md). Only shell integration tests cover this subsystem.
6. **Pre-11.2 compatibility**: `locate_class_for_all_users` does a full heap scan of `_db_user` to resolve unqualified class names from old unload files; ambiguous matches (same table in multiple schemas) are reported as an error.
