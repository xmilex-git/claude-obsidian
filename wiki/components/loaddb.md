---
type: component
parent_module: "[[modules/src|src]]"
path: "src/loaddb/"
status: active
purpose: "Bulk loader for the loaddb utility — bison/flex grammar for CUBRID's own data format, parallel batch processing, direct heap/index insertion bypassing normal SQL execution"
key_files:
  - "load_grammar.yy (bison LALR(1) grammar for loaddb input format)"
  - "load_lexer.l (flex C++ lexer)"
  - "load_driver.cpp (orchestrates scanner + parser; entry point per batch)"
  - "load_session.cpp (session lifecycle, worker dispatch, ordered batch-commit)"
  - "load_worker_manager.cpp (thread pool of driver instances)"
  - "load_server_loader.cpp (server_class_installer, server_object_loader — direct heap insert)"
  - "load_db_value_converter.cpp (string → DB_VALUE type dispatch table)"
  - "load_error_handler.cpp (per-line error reporting, syntax-check mode)"
  - "load_class_registry.cpp (class_entry, attribute metadata cache per session)"
  - "load_common.hpp (batch/class_id types, batch packable object)"
  - "load_object.c (legacy SA object construction)"
  - "load_sa_loader.cpp (SA_MODE entry; still used for standalone)"
namespace: "cubload"
tags:
  - component
  - cubrid
  - loaddb
  - bulk-loader
  - bison
  - flex
related:
  - "[[modules/src|src]]"
  - "[[components/loaddb-grammar|loaddb-grammar]]"
  - "[[components/loaddb-executor|loaddb-executor]]"
  - "[[components/loaddb-driver|loaddb-driver]]"
  - "[[components/parser|parser]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/btree|btree]]"
  - "[[components/thread|thread]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/loaddb/` — Bulk Loader

The `loaddb` utility ingests large data volumes into CUBRID without going through the normal SQL parser or query executor. It has its own bison/flex grammar, its own parallel worker infrastructure, and inserts rows directly into the heap via `locator_insert_force` / `locator_multi_insert_force`.

All code lives in the `cubload` C++ namespace; C-style legacy files (`load_object.c`, `load_object_table.c`) predate the C++ rewrite and are used primarily in SA mode.

## Architecture overview

```
loaddb utility (executables/loaddb.c)
  │
  ├── SA_MODE path ──► load_sa_loader.cpp ──► load_driver → grammar → load_object.c
  │
  └── CS_MODE path ──► network (client sends batches)
                           │
                    cub_server receives
                           │
                    load_session.cpp
                      (session)
                           │
                    load_worker_manager.cpp
                      (worker pool, driver pool)
                           │
                    load_task::execute()
                      per batch, per thread
                           │
                    load_driver.cpp
                      driver::parse(batch content)
                           │
                  load_grammar.yy / load_lexer.l
                           │
                  load_server_loader.cpp
                    server_class_installer  ← %class directive
                    server_object_loader    ← data rows
                           │
                  heap_attrinfo_set()
                  heap_attrinfo_transform_to_disk_except_lob()
                  locator_multi_insert_force()   ← bulk heap insert
                           │
                  xtran_server_commit()
```

## How it differs from SQL INSERT

| Dimension | Normal INSERT | loaddb |
|-----------|--------------|--------|
| Parser | `csql_grammar.y` → `PT_NODE` | `load_grammar.yy` — own LALR(1) grammar |
| Semantic check | `semantic_check.c`, `type_checking.c` | `load_db_value_converter.cpp` per attribute |
| Execution path | `XASL_NODE` → `qexec_execute_mainblock` | `locator_multi_insert_force` / `locator_insert_force` |
| Indexing | Normal index maintenance per row | BU (bulk-update) lock; B-tree updated as records are inserted |
| Locking | Row-level MVCC | Class-level `BU_LOCK` held for entire batch |
| Transaction | Normal txn | One transaction per batch; ordered commit |
| Parallel | No (single-threaded per statement) | Multiple worker threads, each with its own `driver` instance |

## Input file format (grammar)

```
%id table_name 1
%class table_name (col1 col2 col3)
1 'hello' 3.14
2 'world' 2.71
@3:1  -- OID reference
{1 2 3}  -- set literal
```

Key tokens/directives:
- `%class` — specifies target table and optional column list (calls `server_class_installer::install_class`)
- `%id` — assigns a class ID number (calls `check_class`)
- Data rows — space-separated constants, one per line; each row calls `server_object_loader::process_line` then `finish_line`
- Supported constant types: `LDR_NULL`, `LDR_INT`, `LDR_FLOAT`, `LDR_DOUBLE`, `LDR_NUMERIC`, `LDR_STR`, `LDR_DATE`, `LDR_TIME`, `LDR_TIMESTAMP`, `LDR_TIMESTAMPLTZ`, `LDR_TIMESTAMPTZ`, `LDR_DATETIME`, `LDR_DATETIMELTZ`, `LDR_DATETIMETZ`, `LDR_BSTR`, `LDR_XSTR`, `LDR_ELO_INT`, `LDR_ELO_EXT`, `LDR_MONETARY`, `LDR_COLLECTION`, OID references
- Currency symbols are first-class tokens (YEN, WON, DOLLAR, EURO, etc.)

Full grammar detail: [[components/loaddb-grammar|loaddb-grammar]].

## Batch / commit semantics

The file is split into **batches** on the client side. Each batch is a `cubload::batch` (a `packable_object` with a `batch_id`, `class_id`, string content, and line offset). In CS mode:

1. Client sends batches over the network.
2. `load_session` dispatches each batch as a `load_task` to a worker thread.
3. Each worker gets its own `driver` instance from `resource_shared_pool<driver>` (pool size = worker count).
4. Each batch runs inside its own transaction (`logtb_assign_tran_index` + `xtran_server_commit`).
5. **Ordered commit**: `session::wait_for_previous_batch(batch_id)` blocks until batch N-1 has committed before batch N commits — preserving commit order even with parallel parsing.
6. On any batch failure: `session::fail()` is called and `xtran_server_abort` rolls back; subsequent tasks early-exit.

> [!key-insight] Parallelism is in parsing + heap-building; commit is serialised
> Workers parse and accumulate `RECDES` objects in parallel. The flush (`locator_multi_insert_force`) and commit are per-batch but ordered via the condition variable in `session::wait_for_previous_batch`.

## Direct heap insertion path

Inside `server_object_loader::finish_line()`:
1. `heap_attrinfo_transform_to_disk_except_lob()` — serialize the `HEAP_CACHE_ATTRINFO` to a `record_descriptor`.
2. The `RECDES` is pushed into `m_recdes_collected`.

At end-of-batch, `flush_records()` is called:
- Normal path: `locator_multi_insert_force()` — inserts the whole batch of records at once using `MULTI_ROW_INSERT` op type. Wrapped in a system operation (`log_sysop_start` / `log_sysop_attach_to_outer`).
- HA or filtered-error path: falls back to per-record `locator_insert_force()` with individual sysops (so HA replication gets per-row LSAs).

The class holds a `BU_LOCK` (Bulk-Update lock) on the class OID for the entire batch lifetime, checked with `lock_has_lock_on_object`.

## Error model

`load_error_handler` wraps two modes:
- **Syntax-check mode** (`--check-syntax`): errors are non-fatal; rows are counted but not inserted. `on_error_with_line` is used instead of `on_failure_with_line`.
- **Normal mode**: any per-batch error calls `session::fail()`, which propagates to all subsequent workers.

Per-row errors are tracked with `m_current_line_has_error`; if set, `finish_line()` discards the accumulated `RECDES` for that row rather than inserting it. Filtered errors (from `--ignore-error` flags) allow insertion to continue past individual rows.

Error messages come from the CUBRID message catalog (`MSGCAT_CATALOG_UTILS`, `MSGCAT_UTIL_SET_LOADDB`).

## Backwards compatibility (pre-11.2 unload files)

Old `unloaddb` output (version < 11.2) did not include schema-qualified names. `server_class_installer::locate_class` handles this:
- If class name has no dot: prepend the session's user name and retry with `xlocator_find_class_oid`.
- If still not found and client type is `DB_CLIENT_TYPE_ADMIN_LOADDB_COMPAT_UNDER_11_2`: scan all users via `locate_class_for_all_users` (heap scan of `_db_user`). If the name matches exactly one user's table, use it; if ambiguous (two users own same table name), return `LC_CLASSNAME_DELETED` (error).

## Modes

| Mode | Entry | Characteristics |
|------|-------|----------------|
| SA (`SA_MODE`) | `load_sa_loader.cpp` | Direct file access; single process; uses `load_object.c` for legacy object construction |
| CS (`CS_MODE`) | `load_session.cpp` via network | Batches sent over wire; parallel workers; `server_class_installer` + `server_object_loader` |

## Locking convention

Class `BU_LOCK` is acquired before any batch starts and is held throughout. This prevents concurrent DDL but allows concurrent DML from other connections (BU lock is compatible with S and IS locks but conflicts with X). The assertion `lock_has_lock_on_object(&class_oid, oid_Root_class_oid, BU_LOCK)` is checked inside `server_object_loader::init()`.

## Unit tests

> [!warning] LOADDB unit tests are disabled
> The `LOADDB` module under `unit_tests/` is disabled due to compilation issues (noted in the project AGENTS.md). This means the grammar/lexer changes have no automated regression coverage from unit tests. Integration-level shell tests in `tests/` are the only automated coverage.

## Function/class prefix conventions

| Prefix | Role |
|--------|------|
| `cubload::driver` | Orchestrates scanner + parser per batch |
| `cubload::session` | Session-level state, worker dispatch, stats |
| `cubload::server_class_installer` | Processes `%class` directive, registers class in `class_registry` |
| `cubload::server_object_loader` | Processes data rows, accumulates RECDES, calls heap insert |
| `cubload::error_handler` | Per-line error tracking, syntax-check vs. failure mode |
| `ldr_*` | Legacy C-style functions (SA_MODE, `load_object.c`) |
| `load_*` (free functions) | Shared utilities and worker-manager public API |

## Related

- [[components/loaddb-grammar|loaddb-grammar]] — grammar and lexer deep dive
- [[components/loaddb-executor|loaddb-executor]] — heap insert path and server_object_loader
- [[components/loaddb-driver|loaddb-driver]] — driver, session, worker-manager
- [[components/parser|parser]] — contrasting component: SQL grammar (bison/flex), same toolchain
- [[components/heap-file|heap-file]] — `heap_attrinfo_*`, `locator_insert_force` implementations
- [[components/btree|btree]] — index maintenance triggered by `locator_insert_force`
- [[components/thread|thread]] — worker pool infrastructure reused here
- [[Memory Management Conventions]]
- [[Build Modes (SERVER SA CS)]]
- Source: [[sources/cubrid-src-loaddb|cubrid-src-loaddb]]
