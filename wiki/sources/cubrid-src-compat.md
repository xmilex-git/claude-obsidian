---
type: source
title: "CUBRID src/compat/ — Client API & DB_VALUE"
date: 2026-04-23
source_type: codebase
path: "src/compat/"
tags:
  - source
  - cubrid
  - compat
  - db-value
  - client-api
related:
  - "[[components/compat|compat]]"
  - "[[components/db-value|db-value]]"
  - "[[components/client-api|client-api]]"
  - "[[components/dbi-compat|dbi-compat]]"
---

# CUBRID `src/compat/` — Client API & DB_VALUE

Source ingestion date: 2026-04-23. Files read: `AGENTS.md`, `dbtype_def.h`, `dbi_compat.h`, `dbtype_function.h`, `db_macro.c` (top 150 lines), `db_query.c` (top 80 lines), `db_admin.c` (top 60 lines).

## What this directory is

`src/compat/` is CUBRID's public API boundary layer. It owns:
1. **`DB_VALUE`** — the universal tagged-union SQL value container used everywhere in the engine (client and server).
2. **`db_*` client API** — connection, transaction, schema DDL, object CRUD, query compile/execute/fetch, set operations, LOB, JSON.
3. **`dbi_compat.h`** — the combined public header installed with the client library; the anchor of the 6-place error-code rule.

## Key findings

### DB_VALUE structure
- Three fields: `DB_DOMAIN_INFO domain` (type tag + null flag + precision/scale/collation), `DB_DATA data` (union of 20+ typed slots), `need_clear_type need_clear` (ownership flag).
- `DB_TYPE` enum has 41 values, numbered 0–40, stable ABI — values appear on disk and in XASL streams.
- `DB_CHAR` (string member of `DB_DATA`) uses a three-way `style` discriminant: `SMALL_STRING` (inline buf, no heap), `MEDIUM_STRING` (pointer + optional compressed_buf), `LARGE_STRING` (indirect `DB_LARGE_STRING*`).
- `DB_TYPE_NCHAR_DEPRECATED (26)` and `DB_TYPE_VARNCHAR_DEPRECATED (27)` slots are retained for ABI but must not be used in new code.
- `need_clear = true` means `db_value_clear()` must free heap data (string copies, sets, JSON). Most `db_make_*` functions store a pointer and set `need_clear = false`; `db_make_string_copy` is the allocating exception.

### `db_make_*` / `db_get_*` in `db_macro.c`
- All constructors follow: set `domain.general_info.type`, set `domain.general_info.is_null = 0`, write `data.<field>`, set `need_clear`.
- `db_value_put_null`: sets `is_null = 1`, `need_clear = false` — type tag preserved.
- `db_value_domain_init` initializes type + precision/scale without touching `data`.
- `pr_clear_value()` is a server-side alias for `db_value_clear()`.

### Client API families
- `db_admin.c`: lifecycle (login/restart/shutdown), transaction (commit/abort/savepoint/2PC), isolation, volumes, authorization, serials.
- `db_obj.c`: object CRUD — `db_get/put/create/drop`, `db_find_unique`, `db_send` (method call).
- `db_query.c`: `db_compile_statement` → `db_execute_statement` → `db_query_next_tuple` → `db_query_get_tuple_value` → `db_query_end`. Backed by client-side `Qres_table` (global query result registry with pool of `DB_QUERY_RESULT` structs).
- `db_elo.c`: BLOB/CLOB external storage; `DB_ELO` holds a locator string, not inline data.

### `dbi_compat.h`
- Umbrella header; pulls in `dbtype_def.h`, `error_code.h`, `dbtype_function.h`, `db_date.h`, `db_elo.h`, `cache_time.h`.
- `SQLX_CMD_*` macro layer: ~50 `#define` aliases mapping legacy names → `CUBRID_STMT_*` enum values.
- Error codes reach client code via the `error_code.h` include — `dbi_compat.h` is "place 2" in the 6-place rule.
- CCI submodule (`cubrid-cci/src/base_error_code.h`) is place 6; it is an independent copy.

### SA_MODE vs CS_MODE
- `db_admin.c` uses `#if defined(SA_MODE)` guards for direct-call paths (e.g., `pl_sr.h` for stored procedures).
- In CS_MODE: `network_interface_cl.c` handles all server communication.
- Both modes compile the same `db_*` API surface.

## Pages created

- [[components/compat]] — hub page
- [[components/db-value]] — DB_VALUE deep dive
- [[components/client-api]] — db_* function families
- [[components/dbi-compat]] — umbrella header + error-code mirror

## Cross-references noted

- [[components/parser]]: `PT_VALUE` nodes embed `DB_VALUE` for SQL literals
- [[components/regu-variable]]: `REGU_VARIABLE` with `TYPE_DBVAL` holds `DB_VALUE` literals in XASL
- [[components/query-executor]]: `fetch_val_list` evaluates `DB_VALUE` in server-side XASL execution
- [[Error Handling Convention]]: `dbi_compat.h` is place 2 of the 6-place rule
- [[modules/cubrid-cci|cubrid-cci]]: holds the third copy of error codes
