---
created: 2026-04-27
type: source
title: "CUBRID Manual — Stored Procedures (Java SP, methods, cross-cutting)"
source_path: "/home/cubrid/cubrid-manual/en/pl/index.rst, pl_create.rst, pl_call.rst, pl_auth.rst, pl_tcl.rst, pl_tuning.rst, pl_package.rst, jsp.rst, method.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - pl
  - sp
  - java-sp
  - jsp
  - methods
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-plcsql]]"
  - "[[sources/cubrid-src-sp]]"
  - "[[components/sp]]"
  - "[[components/sp-jni-bridge]]"
  - "[[components/sp-method-dispatch]]"
  - "[[components/method]]"
---

# CUBRID Manual — Stored Procedures

**Ingested:** 2026-04-27
**Source files (cross-cutting + Java + legacy methods):** `pl/index.rst` (58), `pl_create.rst` (555), `pl_call.rst` (142), `pl_auth.rst` (104), `pl_tcl.rst` (89), `pl_tuning.rst` (208), `pl_package.rst` (150), `jsp.rst` (1381), `method.rst` (54). Total ~3420 lines.
**Companion page:** [[sources/cubrid-manual-plcsql]] for the PL/CSQL language itself (the four `plcsql_*.rst` files).

## What This Covers

The **engine-side, language-agnostic stored-routine surface**: how to CREATE/CALL/GRANT, package management, transaction control, tuning, plus the **Java SP** runtime and the **deprecated legacy methods** system.

CUBRID supports **two SP languages**: PL/CSQL (Oracle PL/SQL-compatible, new in 11.4) and Java. They share the catalog, calling convention, authorization model, and runtime — but diverge on syntax, type system, and transaction semantics.

## Section Map

| File | Lines | Content |
|---|---|---|
| `index.rst` | 58 | Chapter intro; SP vs SF distinction; toctree ordering |
| `pl_create.rst` | 555 | CREATE PROCEDURE / CREATE FUNCTION cross-language reference; param modes IN/OUT/IN OUT (max 64); type-support matrix PL/CSQL vs Java SP; default-value rules; **no overloading**; object-dependency check (Static SQL register-time, Dynamic SQL execute-time); charset rules |
| `pl_call.rst` | 142 | CALL statement vs SELECT-as-function; **16-call recursion limit when via query** |
| `pl_auth.rst` | 104 | GRANT/REVOKE EXECUTE; **WITH GRANT OPTION not supported**; AUTHID DEFINER/OWNER (synonyms) vs CURRENT_USER/CALLER (synonyms); PL/CSQL only supports Owner's Rights; Owner's Rights net-new in 11.4 |
| `pl_tcl.rst` | 89 | Transaction control inside SPs; **PL/CSQL always honors COMMIT/ROLLBACK; Java SP ignores unless `pl_transaction_control=yes` (default `no`)** |
| `pl_tuning.rst` | 208 | Cost of SF in queries; **DETERMINISTIC for FUNCTIONs only**; SUBQUERY_CACHE trace |
| `pl_package.rst` | 150 | DBMS_OUTPUT — only system package as of 11.4; ENABLE buffer ≤32767 bytes; DISABLE/PUT/PUT_LINE/NEW_LINE/GET_LINE/GET_LINES |
| `jsp.rst` | 1381 | Java SP reference — `public static`, JAVA NAME call-spec, server-side JDBC (`jdbc:default:connection:`), supported/unsupported JDBC APIs, ResultSet auto-close at SP end, OUT params as 1D/2D arrays, RETURN CURSOR, recursion limit 16, JNI via `loadjava -j` |
| `method.rst` | 54 | Legacy OO methods — explicitly **not recommended** for user code; only used by built-ins (e.g., `CALL login(...) ON CLASS db_user`) |

## Key Facts

### CREATE PROCEDURE / FUNCTION
- Identifier max **222 bytes** (post-11.2 schema prefix).
- Parameter modes: IN (default), OUT, IN OUT. **Max 64 params**.
- **`OR REPLACE` silently overwrites** even with different param list — there is **no function overloading** in CUBRID.
- **Default values**: only literals + a closed list of system functions (SYS_TIMESTAMP, UNIX_TIMESTAMP, USER, TO_CHAR, etc.). Stored as **string up to 255 bytes** in `_db_stored_procedure_args`. **User functions forbidden as defaults**.

### Type support matrix (PL/CSQL vs Java SP)
- **Both unsupported**: BIT, ENUM, BLOB, CLOB, JSON, timezone-aware timestamps (TIMESTAMPTZ/LTZ, DATETIMETZ/LTZ, TIMETZ).
- **PL/CSQL also unsupported**: Collections (SET/MULTISET/LIST), CURSOR.
- **Java SP supports**: Collections (as 2D arrays), CUBRIDOID arrays.

### Calling — 16-call recursion limit
- Limit is **16 calls when recursion goes through a SQL query** (e.g., `SELECT my_func(...) INTO k`). Direct recursion (`RETURN n * factorial(n-1)`) has no SP-level limit (only data overflow eventually stops it).
- **Same limit for PL/CSQL and Java SP**.
- Error: `Too many nested stored procedure call`.
- Wiki cross-ref: [[components/sp-method-dispatch]] documents `METHOD_MAX_RECURSION_DEPTH = 15`. The "depth 15 vs count 16" off-by-one is real.

### OUT/IN OUT restriction
- SPs/functions with OUT or IN OUT params **cannot be called from inside a SELECT/UPDATE/DELETE** — only via CALL.
- Error: `Semantic: Stored procedure/function 'X' has OUT or IN OUT arguments`.

### Authorization (`pl_auth.rst`)
- `GRANT EXECUTE ON PROCEDURE <name> TO <user>;`
- `REVOKE EXECUTE ON PROCEDURE <name> FROM <user>;`
- **WITH GRANT OPTION not supported** — execute privilege cannot be re-delegated.
- AUTHID:
  - `DEFINER` / `OWNER` (synonyms) — Owner's Rights, **default**, **PL/CSQL only supports this**, NEW in 11.4.
  - `CURRENT_USER` / `CALLER` (synonyms) — Caller's Rights, **Java SP only**.

### Transaction control (`pl_tcl.rst`)
- **PL/CSQL: COMMIT/ROLLBACK always honored.**
- **Java SP: COMMIT/ROLLBACK ignored unless `pl_transaction_control=yes`** in `cubrid.conf`. Default `no` for backward compat.
- Auto-commit is **always disabled inside any SP**, regardless of session setting.

### Performance (`pl_tuning.rst`)
- **DETERMINISTIC** flag (in CREATE FUNCTION) is FUNCTIONs only — enables correlated-subquery cache hits.
- Trace shows `SUBQUERY_CACHE` line (`hit/miss/size/status: enabled`).
- Built-in vs UDF: ~15× slowdown for trivial UDF wrappers (worked example).

### DBMS_OUTPUT (the only system package)
- `DBMS_OUTPUT.ENABLE(size)` — max 32767 bytes.
- `DISABLE`, `PUT`, `PUT_LINE`, `NEW_LINE`, `GET_LINE(line OUT, status OUT)`, `GET_LINES`.
- CSQL `;server-output on` is internally `DBMS_OUTPUT.ENABLE(20000)`.

## Java SP (jsp.rst)

### Method spec
- Method must be `public static`.
- JAVA NAME format: `'ClassName.methodName(java.lang.String) return java.lang.Type'`.
- Same identifier/param caps as PL/CSQL.

### Server-side JDBC
- URL: **`jdbc:default:connection:`** (note trailing colon). No `Class.forName` required — driver is implicit.
- Supported APIs: Statement, PreparedStatement, CallableStatement, ResultSet, ResultSetMetaData, Connection, DriverManager, SQLException, CUBRIDOID.
- **Unsupported**: DatabaseMetaData, BLOB/CLOB, ParameterMetaData, Savepoint, SQLData, Struct, Array, Ref.

### ResultSet semantics
- Forward-only, read-only, non-sensitive.
- **Auto-closed at SP end** — no holdability across the SP boundary.
- `getHoldability()` lies and returns `HOLD_CURSORS_OVER_COMMIT` but the cursor is gone post-call.

### OUT params from Java
- Primitives: 1-D arrays (`int[]`, `String[]`).
- Collection types: 2-D arrays.
- OID type: `CUBRIDOID[]`.

### Returning ResultSet
- `RETURN CURSOR` Java syntax — Java method returns `ResultSet`, CUBRID exposes as cursor to caller.

### Connection.commit / rollback
- **No-ops** by default. Transaction APIs ignored unless `pl_transaction_control=yes`.

### External-database connections
- If Java SP opens connection to ANOTHER CUBRID DB (not `jdbc:default:connection:`), **must explicitly close** — won't auto-commit on SP exit.

### JNI escape hatch (NEW in 11.4)
- Java SP that calls `System.load()` for native libs must be registered with **`loadjava -j`** (or `--jni`).
- Native lib path: `$CUBRID/jni/`.
- Without `-j`, execution fails with `Library load not allowed`.
- `-j` flag is **sticky** — requires PL server restart when changed.

## Legacy methods (method.rst)

- C-implemented methods callable via `CALL <method> [ON CLASS] <target>`.
- **Explicitly deprecated** in the manual: *"User-defined methods are not recommended due to many constraints and are only used in system-built methods."*
- Built-in survivor: `CALL login('U', '') ON CLASS db_user` (the auth method).

## Cross-References

- [[sources/cubrid-manual-plcsql]] — the PL/CSQL language reference (companion)
- [[sources/cubrid-src-sp]] — `src/sp/` source ingest (JNI bridge, catalog, lifecycle)
- [[components/sp]] — SP hub
- [[components/sp-jni-bridge]] — invocation mechanics, DB_VALUE marshalling
- [[components/sp-method-dispatch]] — XASL → executor dispatch, recursion enforcement
- [[components/sp-protocol]] — transport, SP_CODE opcodes
- [[components/method]] — legacy method system

## Incidental Wiki Enhancements

- [[components/sp]]: documented `pl_transaction_control` (default `no`) — Java SP COMMIT/ROLLBACK ignored unless set; PL/CSQL always honors; SPs always run with auto-commit disabled regardless of caller session.
- [[components/sp]]: documented DETERMINISTIC = FUNCTION-only flag, ties into subquery_cache trace; default-arg storage in `_db_stored_procedure_args` capped at 255-byte string with closed-list of allowed system functions; `STORED_PROCEDURE_RETURN_NUMERIC_SIZE` controls NUMERIC return precision (default `(38, 15)`).
- [[components/sp-method-dispatch]]: documented the 16-call recursion limit applies only when recursion goes through a SQL query; direct recursion has no SP-level limit. Reconciled with code's `METHOD_MAX_RECURSION_DEPTH = 15` — depth-vs-count off-by-one.
- [[components/sp-method-dispatch]]: documented OUT/IN OUT parameter restriction — SPs with OUT/INOUT cannot be called from SELECT/UPDATE/DELETE, only via CALL.
- [[components/sp-jni-bridge]]: documented server-side JDBC ResultSet lifecycle (forward-only, read-only, auto-close at SP end, `getHoldability()` lies); `loadjava -j` (NEW 11.4) required for Java SPs that call `System.load()` for native libs; `-j` is sticky and requires PL server restart.
- [[components/method]]: added "deprecated for user code" stance citing manual's `method.rst:13-14`.

## Key Insight

The two SP languages share machinery but diverge on critical semantics: **PL/CSQL always honors COMMIT/ROLLBACK; Java SP needs `pl_transaction_control=yes`**. This single parameter is the most common "Java SP doesn't commit" trap. **No overloading** means `CREATE OR REPLACE` is dangerous — silently overwrites by name regardless of param list. **The 16-call recursion limit applies only to SQL-mediated recursion** — pure recursive Java methods can exceed it freely.
