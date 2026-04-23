---
type: component
parent_module: "[[modules/src|src]]"
path: "src/compat/"
status: active
purpose: "Public client API surface (db_* namespace) and the universal value container DB_VALUE; bridges client and server sides; stability-critical — these are the ABI-facing headers"
key_files:
  - "dbtype_def.h (DB_VALUE struct, DB_TYPE enum, all supporting type definitions — central)"
  - "dbi_compat.h (client-visible API declarations + SQLX_CMD_* alias macros + error-code mirror)"
  - "dbtype_function.h (DB_MAKE_*/DB_GET_* macro layer + db_value_* function declarations)"
  - "db_macro.c (db_make_*() and db_get_*() implementations — ~1 K lines)"
  - "db_query.c (client-side query execution API: compile, execute, fetch, cursor)"
  - "db_admin.c (database lifecycle: login, restart, shutdown, add_volume, stats)"
  - "db_obj.c (object-level API: create, get, put, drop, find_unique, send)"
  - "db_vdb.c (virtual database / view support API)"
  - "db_set.c (set/multiset/sequence DB_VALUE operations)"
  - "db_date.c (date/time encode-decode and DB_VALUE helpers)"
  - "db_json.cpp (JSON DB_VALUE support)"
  - "db_elo.c (LOB / External Large Object API)"
  - "db_value_printer.cpp (DB_VALUE → human-readable string)"
  - "db_set_function.h (set operation function declarations)"
public_api:
  - "db_make_int / db_make_string / db_make_null / … (construct DB_VALUE)"
  - "db_get_int / db_get_string / … (read DB_VALUE)"
  - "db_value_clear / db_value_free / db_value_clone (lifecycle)"
  - "db_value_domain_type / db_value_is_null / DB_IS_NULL / DB_VALUE_DOMAIN_TYPE (inspect)"
  - "db_login / db_restart / db_restart_ex / db_shutdown (connection)"
  - "db_commit_transaction / db_abort_transaction / db_savepoint_transaction (transaction)"
  - "db_compile_statement / db_execute_statement (query execution)"
  - "db_get / db_put / db_create / db_drop (object API)"
  - "db_error_code / db_error_string (error inspection)"
tags:
  - component
  - cubrid
  - compat
  - db-value
  - client-api
related:
  - "[[modules/src|src]]"
  - "[[components/db-value|db-value]]"
  - "[[components/client-api|client-api]]"
  - "[[components/dbi-compat|dbi-compat]]"
  - "[[components/parser|parser]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/query-executor|query-executor]]"
  - "[[Error Handling Convention]]"
  - "[[modules/cubrid-cci|cubrid-cci]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/compat/` — Client API & DB_VALUE

The `compat/` layer is CUBRID's public-facing boundary. It defines everything an application or embedded client sees: the `db_*` function namespace, the `DB_VALUE` universal value container, and the `dbi_compat.h` combined public header.

> [!key-insight] Stability contract
> `dbi_compat.h` and `dbtype_def.h` are installed with the CUBRID client libraries. Changing a struct field, reordering `DB_TYPE` enum values, or renaming a `db_*` function is a binary-compatibility break. Treat these files with more caution than any internal header.

## Sub-components

| Sub-page | What it covers |
|----------|---------------|
| [[components/db-value\|db-value]] | `DB_VALUE` struct anatomy, `DB_TYPE` enum, `db_make_*` / `db_get_*` patterns, `need_clear` lifetime |
| [[components/client-api\|client-api]] | `db_*` function families: connection, transaction, schema, query, object, OID |
| [[components/dbi-compat\|dbi-compat]] | `dbi_compat.h` role as the combined public header; `SQLX_CMD_*` alias layer; error-code mirror |

## File map

| File | Role |
|------|------|
| `dbtype_def.h` | `DB_VALUE` union + `DB_TYPE` enum + all supporting types — pulled into `dbi_compat.h` |
| `dbi_compat.h` | Umbrella public header: includes `dbtype_def.h`, `dbtype_function.h`, `error_code.h`, `db_date.h`, `db_elo.h`, `cache_time.h` |
| `dbtype_function.h` | `DB_MAKE_*` / `DB_GET_*` macro layer + `extern` declarations for `db_value_*` functions |
| `db_macro.c` | `db_make_*()` and `db_get_*()` implementations; also `db_value_domain_init`, `db_value_clear`, coercion helpers |
| `db_query.c` | Client-side query lifecycle: `db_compile_statement`, `db_execute_statement`, `db_query_next_tuple`, `db_query_end` |
| `db_admin.c` | Database lifecycle: `db_login`, `db_restart`, `db_shutdown`, `db_add_volume`, `db_num_volumes` |
| `db_obj.c` | Object-level API: `db_create`, `db_get`, `db_put`, `db_drop`, `db_find_unique`, `db_send` |
| `db_vdb.c` | Virtual database / view support |
| `db_set.c` | `DB_SET`, `DB_MULTISET`, `DB_SEQ` construction and element manipulation |
| `db_date.c` | `DB_DATE`, `DB_TIME`, `DB_TIMESTAMP`, `DB_DATETIME` encode/decode helpers |
| `db_json.cpp` | JSON DB_VALUE: `JSON_DOC` wrapping, deep copy, scalar conversion |
| `db_elo.c` | LOB / ELO API: read, write, size, copy, delete for external large objects |
| `db_value_printer.cpp` | `db_value_fprint` — type-dispatch to human-readable string |
| `db_set_function.h` | Set operation function declarations |

## Wire position

```
Client application / CSQL / broker CAS
         │
         ▼
  dbi_compat.h ─── db_*.c (compat layer)
         │
         ├──► object/         (schema, auth, workspace)
         ├──► parser/         (SQL text → PT_NODE → XASL)
         └──► network_interface_cl / transaction_cl
                    │
                    ▼
              cub_server (SERVER_MODE)
```

The `db_*` functions sit entirely client-side (`CS_MODE` or `SA_MODE`). They delegate to:
- `object/` for schema and object management
- `parser/` for SQL compilation
- `transaction_cl.c` for commit/abort/savepoint
- `network_interface_cl.c` for RPC to `cub_server`

In `SA_MODE`, network calls are replaced by direct in-process calls.

## Naming conventions

| Prefix | Meaning |
|--------|---------|
| `db_make_*` | Construct a `DB_VALUE` from a C scalar |
| `db_get_*` | Read a C scalar from a `DB_VALUE` |
| `db_value_*` | Lifecycle/inspection operations on `DB_VALUE` |
| `db_*` (other) | Client API functions (connection, query, schema, etc.) |
| `pr_*` | Primitive (server-internal); alias for some `db_value_*` ops |
| `DB_MAKE_*` / `DB_GET_*` | Macro aliases for the above (backward compatibility) |

## Gotchas

> [!warning] `need_clear` ownership
> `db_make_string` stores a **pointer** — no copy. `db_make_string_copy` copies. `db_make_varchar` / `db_make_char` also take a pointer. Always check whether a specific `db_make_*` copies or borrows; the wrong assumption silently leaks or double-frees.

> [!warning] `DB_TYPE_NCHAR_DEPRECATED` / `DB_TYPE_VARNCHAR_DEPRECATED`
> Enum slots 26 and 27 are preserved for ABI compatibility only. New code must not create new paths for these types; the grammar maps NCHAR to `DB_TYPE_CHAR` via `#define`.

> [!warning] `DB_TYPE` numeric values are ABI
> `DB_TYPE` enum values are embedded in on-disk page headers and serialized XASL streams. Never renumber or remove existing enum values.

## Related

- [[components/db-value|db-value]] — DB_VALUE deep dive
- [[components/client-api|client-api]] — full `db_*` function catalog
- [[components/dbi-compat|dbi-compat]] — error-code mirror and umbrella header
- [[components/parser|parser]] — uses DB_VALUE for PT_VALUE literal nodes
- [[components/regu-variable|regu-variable]] — REGU_VARIABLE wraps / produces DB_VALUE
- [[components/query-executor|query-executor]] — evaluates DB_VALUE in server-side XASL
- [[Error Handling Convention]] — the 6-place rule; `dbi_compat.h` is place 2
- [[modules/cubrid-cci|cubrid-cci]] — CCI driver mirrors error codes a third time
- Source: [[sources/cubrid-src-compat|cubrid-src-compat]]
