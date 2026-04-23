---
type: component
title: "execute-statement — Server-Side Statement Dispatch (Post-XASL)"
parent_module: "[[modules/src|src]]"
path: src/query/execute_statement.{c,h}
status: developing
key_files:
  - src/query/execute_statement.c
  - src/query/execute_statement.h
public_api:
  - do_statement
  - do_prepare_statement
  - do_execute_statement
  - do_select / do_prepare_select / do_execute_select
  - do_insert / do_prepare_insert / do_execute_insert
  - do_update / do_prepare_update / do_execute_update
  - do_delete / do_prepare_delete / do_execute_delete
  - do_merge / do_prepare_merge / do_execute_merge
  - do_alter / do_create_index / do_alter_index / do_drop_index
tags:
  - cubrid
  - query
  - execution
  - dml
  - ddl
  - client-side
---

# execute-statement — Server-Side Statement Dispatch (Post-XASL)

> [!key-insight]
> Despite the name, `execute_statement.c` runs **client-side** (`#if defined(SERVER_MODE) #error`). It operates on the `PT_NODE` parse tree **after** semantic check and **before** XASL serialization to the server. The file owns the top-level `do_statement` / `do_execute_statement` dispatcher and all DML/DDL prepare+execute helpers.

## Purpose

`execute_statement.c` (~22 K lines) is the **client-side statement execution hub**. It:

1. Accepts a fully name-resolved, semantically checked `PT_NODE *statement`.
2. Routes by `node_type` to the appropriate `do_*` family function (prepare / execute split).
3. For DML (SELECT / INSERT / UPDATE / DELETE / MERGE): calls `do_prepare_select` → optimizer → XASL generation → `prepare_query()` → `execute_query()` → returns a `QFILE_LIST_ID` cursor.
4. For DDL (ALTER, CREATE, DROP, …): calls schema-layer helpers (most in `execute_schema.c`).
5. Fires trigger hooks (`tr_before`/`tr_after`) around DML execution.
6. Handles SERIAL DDL (`do_create_serial`, `do_alter_serial`, `do_drop_serial`).
7. Handles transaction control (`do_commit`, `do_rollback`, `do_savepoint`).
8. Handles synonym DDL, server-link DDL, session variable statements.

---

## Public Entry Points

| Signature | Role |
|-----------|------|
| `int do_statement(PARSER_CONTEXT*, PT_NODE*)` | Single-shot prepare+execute; top-level entry from `db_execute*` path |
| `int do_prepare_statement(PARSER_CONTEXT*, PT_NODE*)` | Prepare phase: builds XASL, registers in server XASL cache |
| `int do_execute_statement(PARSER_CONTEXT*, PT_NODE*)` | Execute phase: sends host vars, gets `QFILE_LIST_ID` |
| `int do_select(PARSER_CONTEXT*, PT_NODE*)` | Calls `do_prepare_select` then `do_execute_select` |
| `int do_prepare_select(PARSER_CONTEXT*, PT_NODE*)` | Optimizer + XASL gen + `prepare_query()` |
| `int do_execute_select(PARSER_CONTEXT*, PT_NODE*)` | `execute_query()` → stores `QFILE_LIST_ID` in `statement->etc` |
| `int do_insert(…)` / `do_prepare_insert(…)` / `do_execute_insert(…)` | INSERT pipeline |
| `int do_update(…)` / `do_prepare_update(…)` / `do_execute_update(…)` | UPDATE pipeline |
| `int do_delete(…)` / `do_prepare_delete(…)` / `do_execute_delete(…)` | DELETE pipeline |
| `int do_merge(…)` / `do_prepare_merge(…)` / `do_execute_merge(…)` | MERGE pipeline |
| `int do_alter(PARSER_CONTEXT*, PT_NODE*)` | ALTER TABLE / VIEW dispatcher |
| `int do_create_index / do_alter_index / do_drop_index` | Index DDL |
| `int do_create_serial / do_alter_serial / do_drop_serial` | SERIAL DDL |
| `int do_check_delete_trigger / do_check_insert_trigger / …` | Trigger pre-check wrappers |
| `int do_replicate_statement(PARSER_CONTEXT*, PT_NODE*)` | CDC/replication supplemental log append |

---

## Statement Kind Branch Table

```
do_statement(parser, stmt)
    └─ switch stmt->node_type
         PT_SELECT / PT_UNION / PT_DIFFERENCE / PT_INTERSECTION
                └─ do_select() → do_prepare_select → do_execute_select
         PT_INSERT    └─ do_insert → do_check_insert_trigger → insert loop
         PT_UPDATE    └─ do_update → do_check_update_trigger → update loop
         PT_DELETE    └─ do_delete → do_check_delete_trigger → delete loop
         PT_MERGE     └─ do_merge → check_merge_trigger → insert/update/delete sub-steps
         PT_CREATE_ENTITY  └─ do_create_entity (execute_schema.c)
         PT_ALTER          └─ do_alter (execute_schema.c)
         PT_CREATE_INDEX / PT_ALTER_INDEX / PT_DROP_INDEX
                           └─ do_create_index / do_alter_index / do_drop_index
         PT_CREATE_SERIAL  └─ do_create_serial
         PT_ALTER_SERIAL   └─ do_alter_serial
         PT_COMMIT         └─ do_commit
         PT_ROLLBACK       └─ do_rollback
         PT_SAVEPOINT      └─ do_savepoint
         PT_DO             └─ do_execute_do
         PT_SET_SESSION_VARIABLES  └─ do_set_session_variables
         ... (40+ total branches)
```

---

## Execution Path — SELECT

```
db_execute_with_result(sql)
    parse → semantic check
    do_prepare_select(parser, stmt)
        SHA1(stmt->alias_print) → sql_hash_text
        prepare_query(context, &stream)          [client→server: cache lookup]
        if cache miss:
            parser_generate_xasl(parser, stmt)   [optimizer+XASL gen]
            xts_map_xasl_to_stream(xasl, &stream) [serialize]
            prepare_query(context, &stream)       [upload to server XASL cache]
        stmt->xasl_id = stream.xasl_id
    do_execute_select(parser, stmt)
        execute_query(xasl_id, &query_id, …, &list_id)
            qmgr_execute_query(…)                [IPC→server]
        stmt->etc = list_id                      [available to cursor layer]
```

---

## Trigger Firing Hook Points

Triggers are fired in a sandwich pattern around the actual DML call:

```
tr_prepare_statement(&state, TR_EVENT_STATEMENT_UPDATE, class_, …)
tr_before(state)                   ← BEFORE STATEMENT triggers fire
    do_check_internal_statements(parser, stmt, do_func)
        do_func(parser, stmt)       ← actual DML
tr_after(state)                    ← AFTER STATEMENT triggers fire
```

`do_Trigger_involved` (global bool) is set during any DML that fires triggers.  
`cdc_Trigger_involved` tracks whether a trigger ran during CDC supplemental logging.

> [!warning]
> `do_Trigger_involved` and `cdc_Trigger_involved` are **global mutable booleans**. They are not thread-safe in CS_MODE (where each CAS thread runs independently). In SA_MODE (single-threaded) this is safe. In CS_MODE the CAS process is single-threaded per connection, so it is also safe per connection — but it is not re-entrant across nested trigger calls without the `CDC_TRIGGER_INVOLVED_BACKUP/RESTORE` macros.

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Build mode | `#if defined(SERVER_MODE) #error` — strictly client (CS_MODE or SA_MODE) |
| Thread safety | Single-threaded per CAS connection; `do_Trigger_involved` / `cdc_Trigger_involved` not re-entrant without macros |
| Memory | PT_NODE trees allocated in `parser_alloc` arena; XASL stream allocated by `xts_map_xasl_to_stream`, freed after `prepare_query` |
| Savepoints | Every DML that can trigger uses a named savepoint (`UtrP`, `UmsP`, etc.) for atomicity with triggers |
| Error propagation | C-style; callers must check return value; parser errors surfaced via `pt_has_error(parser)` |

---

## Lifecycle

```
Per-statement:
  allocate parser context (persistent per session or per statement)
  do_prepare_statement → XASL_ID stored in stmt->xasl_id (reused on re-execute)
  do_execute_statement → QFILE_LIST_ID in stmt->etc
  caller fetches rows via cursor API
  caller calls db_query_end → list file freed
  on transaction commit/rollback: temp files freed via qmgr_clear_trans_wakeup
```

---

## Related

- [[components/query-executor]] — `qexec_execute_mainblock`, called server-side after XASL deserialization
- [[components/query-cl]] — `prepare_query` / `execute_query` thin client wrappers called by this module
- [[components/xasl-generation]] — `parser_generate_xasl` produces the XASL tree
- [[components/xasl-stream]] — `xts_map_xasl_to_stream` / `stx_map_stream_to_xasl`
- [[components/parser]] — PT_NODE parse tree consumed by all `do_*` functions
- [[components/execute-schema]] — `do_alter`, `do_create_entity` DDL bodies live here
- [[components/query-serial]] — serial DDL helpers declared in this header
- [[components/cursor]] — opens cursor over `QFILE_LIST_ID` returned by execute_select
- [[flows/dml-execution-path]] — end-to-end DML flow
- [[flows/ddl-execution-path]] — end-to-end DDL flow
- [[Build Modes (SERVER SA CS)]]
- [[Memory Management Conventions]]
