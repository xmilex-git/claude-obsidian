---
type: component
parent_module: "[[components/compat|compat]]"
path: "src/compat/dbi_compat.h"
status: active
purpose: "Umbrella public header installed with the CUBRID client library; includes all type definitions, function declarations, and SQLX_CMD_* alias macros; also hosts the client-visible copy of error codes that must mirror error_code.h"
key_files:
  - "dbi_compat.h — the combined public header"
  - "src/base/error_code.h — source of truth for error codes (server and client)"
  - "cubrid-cci/src/base_error_code.h — third copy in the CCI submodule"
tags:
  - component
  - cubrid
  - dbi-compat
  - compat
  - error-codes
related:
  - "[[components/compat|compat]]"
  - "[[components/client-api|client-api]]"
  - "[[components/db-value|db-value]]"
  - "[[Error Handling Convention]]"
  - "[[modules/cubrid-cci|cubrid-cci]]"
  - "[[components/error-manager|error-manager]]"
created: 2026-04-23
updated: 2026-04-23
---

# `dbi_compat.h` — Umbrella Public Header

`dbi_compat.h` is the single header that CUBRID client applications and tools include. It bundles all type definitions, function declarations, and macro aliases that form the public C API. It is installed alongside the client shared library (`libcubridcs.so` / `libcubridsa.so`).

## What it includes

```c
/* dbi_compat.h pull chain */
#include "dbtran_def.h"      /* DB_TRAN_ISOLATION enum */
#include "dbtype_def.h"      /* DB_VALUE, DB_TYPE, all type structs */
#include "error_code.h"      /* ER_* constants — duplicated here from base/ */
#include "dbtype_function.h" /* DB_MAKE_*/DB_GET_* macros + db_value_* externs */
#include "db_date.h"         /* db_date_encode, db_time_decode, … */
#include "db_elo.h"          /* db_elo_* LOB API */
#include "cache_time.h"      /* CACHE_TIME for result caching */
```

Everything declared in `db_admin.c`, `db_obj.c`, `db_query.c`, and friends is declared via `extern` in `dbi_compat.h`. Application code `#include "dbi.h"` (which re-exports `dbi_compat.h` via a thin wrapper in `sa/` or `cs/`).

## Error-code mirror

`dbi_compat.h` includes `error_code.h` directly, making all `ER_*` constants visible to client code. This is the mechanism for the **6-place error-code rule**:

> [!key-insight] Six places for a new error code
> When adding a new error code, you must update in this order:
> 1. `src/base/error_code.h` — define `ER_SOME_NEW_ERROR = -NNNN`
> 2. `src/compat/dbi_compat.h` — include path already covers it (via `error_code.h`), but **if you add a client-visible alias or a `#define` rename**, it goes here
> 3. `msg/en_US.utf8/cubrid.msg` — English message text
> 4. `msg/ko_KR.utf8/cubrid.msg` — Korean message text
> 5. Update `ER_LAST_ERROR` in `error_code.h`
> 6. `cubrid-cci/src/base_error_code.h` — CCI submodule copy (if the error is client-facing)

The `dbi_compat.h` itself does not define error codes; it exposes them by including `error_code.h`. The "place 2" designation in the convention refers to verifying the code is reachable via the public header chain.

## `SQLX_CMD_*` alias layer

`dbi_compat.h` defines a complete `#define` alias layer mapping legacy `SQLX_CMD_*` names to the canonical `CUBRID_STMT_*` enum values:

```c
#define SQLX_CMD_TYPE      CUBRID_STMT_TYPE
#define SQLX_CMD_INSERT    CUBRID_STMT_INSERT
#define SQLX_CMD_SELECT    CUBRID_STMT_SELECT
#define SQLX_CMD_UPDATE    CUBRID_STMT_UPDATE
#define SQLX_CMD_DELETE    CUBRID_STMT_DELETE
/* ... ~50 more aliases ... */
#define SQLX_MAX_CMD_TYPE  CUBRID_MAX_STMT_TYPE
```

These exist for backward compatibility with client code predating the `CUBRID_STMT_*` rename. `CUBRID_STMT_TYPE` is the enum defined in `dbtype_def.h`; `SQLX_CMD_*` names predate it.

## `CUBRID_STMT_TYPE` enum

`CUBRID_STMT_TYPE` (in `dbtype_def.h`, exposed via `dbi_compat.h`) classifies the statement type of a compiled SQL statement. It is returned by `db_get_statement_type(session, stmt_id)`:

| Range | Examples |
|-------|---------|
| DDL | `CUBRID_STMT_CREATE_CLASS`, `DROP_CLASS`, `ALTER_CLASS`, `CREATE_INDEX`, … |
| DML | `CUBRID_STMT_INSERT`, `UPDATE`, `DELETE`, `MERGE`, `TRUNCATE` |
| Query | `CUBRID_STMT_SELECT`, `SELECT_UPDATE` |
| Transaction | `CUBRID_STMT_COMMIT_WORK`, `ROLLBACK_WORK`, `SAVEPOINT` |
| Admin | `CUBRID_STMT_CREATE_STORED_PROCEDURE`, `VACUUM`, `KILL`, `SET_TIMEZONE` |

`CUBRID_MAX_STMT_TYPE` is the sentinel count.

## DB_FETCH_MODE constants

`DB_FETCH_MODE` (in `dbtype_def.h`) specifies lock intent when fetching objects:

| Constant | Value | Meaning |
|----------|-------|---------|
| `DB_FETCH_READ` | 0 | Shared read lock |
| `DB_FETCH_WRITE` | 1 | Exclusive write lock |
| `DB_FETCH_DIRTY` | 2 | No lock — potentially stale (internal) |
| `DB_FETCH_CLREAD_INSTREAD` | 3 | Read class + shared instance lock |
| `DB_FETCH_CLREAD_INSTWRITE` | 4 | Read class + exclusive instance (create) |
| `DB_FETCH_QUERY_READ` | 5 | Read class + query-read all instances (SELECT) |
| `DB_FETCH_QUERY_WRITE` | 6 | Read class + update some instances (UPDATE/DELETE) |
| `DB_FETCH_SCAN` | 7 | Read class for index load (lock held for later) |
| `DB_FETCH_EXCLUSIVE_SCAN` | 8 | Exclusive scan for index load |

## Key constants defined here

```c
#define DB_MAX_IDENTIFIER_LENGTH  255    /* max schema object name length */
#define DB_MAX_USER_LENGTH         32    /* max user name */
#define DB_MAX_NUMERIC_PRECISION   38    /* max NUMERIC(p,s) precision */
#define DB_MAX_STRING_LENGTH  0x3fffffff /* max VARCHAR / BIT VARYING length */
#define DB_CURSOR_SUCCESS           0    /* db_query_next_tuple success */
#define DB_CURSOR_END               1    /* no more tuples */
#define DB_CURSOR_ERROR            -1    /* iteration error */
#define DB_TRUE   1
#define DB_FALSE  0
#define DB_EMPTY_SESSION  0              /* uninitialized session id */
```

## Relationship to CCI (`cubrid-cci`)

The CCI (CUBRID C Interface) submodule (`cubrid-cci/`) is a separate client driver with its own copy of error codes in `cubrid-cci/src/base_error_code.h`. This is the **third** location of error codes (after `error_code.h` and the `dbi_compat.h` include chain). CCI does not link against the main engine; it speaks the broker protocol independently.

> [!contradiction] Three copies of error codes
> `error_code.h` (canonical), `dbi_compat.h` chain (indirectly), and `cubrid-cci/src/base_error_code.h` (explicit copy) must all be kept in sync. The AGENTS.md "6-place rule" in the CUBRID root documents this, but CCI is a separate repo with its own release cycle — drift is a known risk. See [[modules/cubrid-cci|cubrid-cci]] and [[Error Handling Convention]].

## Why not one header?

`dbi_compat.h` evolved as the external-facing wrapper around a collection of internal headers. The split (`dbtype_def.h` for types, `dbtype_function.h` for function declarations) allows the engine to include individual headers internally while external clients include only `dbi_compat.h`. This is noted in `dbtype_def.h`:

```c
/* It will be exposed as part of the dbi_compat.h file. */
```

## Related

- Parent: [[components/compat|compat]]
- [[components/client-api|client-api]] — all `db_*` functions declared here
- [[components/db-value|db-value]] — `DB_VALUE` defined in `dbtype_def.h`, pulled in here
- [[components/error-manager|error-manager]] — `er_set` / `er_errid` behind `db_error_code()`
- [[Error Handling Convention]] — the 6-place rule; this header is place 2
- [[modules/cubrid-cci|cubrid-cci]] — CCI submodule with its own error-code copy (place 6)
- Source: [[sources/cubrid-src-compat|cubrid-src-compat]]
