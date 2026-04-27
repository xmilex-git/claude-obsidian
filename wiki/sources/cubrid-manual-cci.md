---
created: 2026-04-27
type: source
title: "CUBRID Manual — CCI Driver (C API)"
source_path: "/home/cubrid/cubrid-manual/en/api/cci.rst, cciapi.rst, cci_index.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - cci
  - api
  - c
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-api]]"
  - "[[sources/cubrid-manual-jdbc]]"
  - "[[modules/cubrid-cci]]"
  - "[[components/cas]]"
  - "[[components/broker-impl]]"
  - "[[components/dbi-compat]]"
  - "[[components/connection]]"
---

# CUBRID Manual — CCI Driver (C API)

**Ingested:** 2026-04-27
**Source files:** `api/cci.rst` (1329), `api/cciapi.rst` (3761), `api/cci_index.rst` (9). Total **5099 lines**.
**Implementation:** lives in submodule `modules/cubrid-cci` (separate from main repo `src/`); pending dedicated source ingest per `hot.md`.

## What CCI Is

The **C-language CUBRID Call Interface** — the foundation of the CUBRID driver ecosystem. All other drivers except ADO.NET and Node.js are CCI wrappers. Defines the wire protocol contract with CAS.

## Library Layout (post-11.2)

| Path | Content |
|---|---|
| `$CUBRID/cci/include/cas_cci.h` | Public header (~95 functions, types, structs, enums) |
| `$CUBRID/cci/lib/libcascci.so` | Linux shared lib |
| `$CUBRID/cci/lib/libcascci.a` | Linux static lib |
| `$CUBRID/cci/bin/cascci.dll` | Windows DLL |

Pre-11.2 paths were different — note the migration in upgrade docs.

## API Surface (95 functions)

### Connect / disconnect
- `cci_connect(host, port, db, user, pw)`
- `cci_connect_ex(host, port, db, user, pw, err)`
- `cci_connect_with_url(url, user, pw)` — most flexible; URL form supports props
- `cci_connect_with_url_ex(url, user, pw, err)`
- `cci_disconnect(conn, err)`
- `cci_init()`

### URL form
```
cci:CUBRID:<host>:<port>:<db_name>:<db_user>:<db_password>:[?<properties>]
```
Properties (case-insensitive, snake_case ↔ camelCase aliases):
- `altHosts`, `loadBalance`, `rcTime` (HA failover)
- `loginTimeout` / `login_timeout`
- `queryTimeout` / `query_timeout`
- `disconnectOnQueryTimeout` (default false; true = close socket on timeout)
- `logFile`, `logBaseDir`, `logSlowQueries`, `slowQueryThresholdMillis`
- `logTraceApi`, `logTraceNetwork`
- `useSSL` (TLS/SSL)

### Prepare / execute / fetch
- `cci_prepare(conn, sql, flag, err)`
- `cci_prepare_and_execute(conn, sql, max_col_size, exec_retval, err)`
- `cci_execute(req, flag, max_col_size, err)`
- `cci_execute_array(req, query_result, err)`
- `cci_execute_batch(conn, num_query, sql_stmts, query_result, err)`
- `cci_execute_result(req, query_result, err)`
- `cci_fetch(req, err)` / `cci_fetch_sensitive` / `cci_fetch_buffer_clear` / `cci_fetch_size`
- `cci_get_data(req, col_no, a_type, value, indicator)`
- `cci_next_result(req, err)` — for `;`-separated multi-statement
- `cci_close_req_handle(req)`

### Cursor
- `cci_cursor(req, offset, position, err)`
- `cci_cursor_update(req, col_no, a_type, value, err)`
- `cci_get_cur_oid(req, oid_str_buf, err)`

### Bind / OID / OUT params
- `cci_bind_param(req, col_no, a_type, value, u_type, flag)` — `CCI_BIND_PTR` for shallow vs deep copy
- `cci_bind_param_array(req, col_no, ...)` — for batch
- `cci_register_out_param(req, col_no)`

### LOB / BLOB / CLOB
- `cci_blob_read`, `cci_blob_size`, `cci_blob_write`, `cci_blob_new`, `cci_blob_free`
- `cci_clob_*` symmetric

### Set / collection
- `cci_set_make`, `cci_set_get`, `cci_set_size`, `cci_set_get_element`, `cci_set_free`

### Transaction
- `cci_end_tran(conn, type=CCI_TRAN_COMMIT|CCI_TRAN_ROLLBACK, err)`
- `cci_savepoint(conn, code, name, err)`
- `cci_get_autocommit(conn)`, `cci_set_autocommit(conn, mode)`
- `cci_get_holdability(conn)`, `cci_set_holdability(conn, mode)`
- `cci_get_isolation_level(conn, level, err)`, `cci_set_isolation_level(conn, level, err)`

### Connection pool (Datasource)
- `cci_property_create()`, `cci_property_set/get`, `cci_property_destroy`
- `cci_datasource_create(props, err)` — creates pool; properties: `pool_size`, `max_wait`, `pool_prepared_statement`, `login_timeout`, `query_timeout`
- `cci_datasource_borrow(ds, err)` / `cci_datasource_release(ds, conn, err)`
- `cci_datasource_change_property(ds, key, val)`
- `cci_datasource_destroy(ds)`
- `CCI_ER_DATASOURCE_TIMEOUT` (-20036) when exhausted

### Schema introspection
- `cci_schema_info(conn, type, class_name, attr_name, flag, err)` — returns a `req_handle`-like result; types via `CCI_SCH_*` constants

### Misc
- `cci_get_db_version(conn, buf, len)`
- `cci_get_query_plan(req, buf)`
- `cci_get_cas_info(conn, buf, len, err)` — CAS_INFO for SQL log lookup
- `cci_get_last_insert_id(conn, value, err)`
- `cci_get_login_timeout` / `cci_set_login_timeout`
- `cci_get_query_timeout` / `cci_set_query_timeout`
- `cci_set_max_row(req, max)`
- `cci_escape_string(conn, dest, src, len, err)`
- `cci_row_count(conn, err)`

## Type System

**Two parallel type universes** — A-types (application/C side) and U-types (DB/server side).

### `T_CCI_A_TYPE` (15 application types)
`CCI_A_TYPE_STR`, `_INT`, `_FLOAT`, `_DOUBLE`, `_BIT`, `_DATE`, `_DATE_TZ` (timezone-aware), `_SET`, `_BLOB`, `_CLOB`, `_REQ_HANDLE`, `_BIGINT`, `_UINT`, `_UBIGINT`, `_NUMERIC`.

### `T_CCI_U_TYPE` (~25 DB types)
Mirrors `DB_TYPE` enum: `CCI_U_TYPE_CHAR`, `_STRING`, `_NCHAR`, `_VARNCHAR`, `_BIT`, `_VARBIT`, `_NUMERIC`, `_INT`, `_SHORT`, `_FLOAT`, `_DOUBLE`, `_MONETARY`, `_DATE`, `_TIME`, `_TIMESTAMP`, `_OBJECT`, `_SET`, `_MULTISET`, `_SEQUENCE`, `_BIGINT`, `_DATETIME`, `_BLOB`, `_CLOB`, `_ENUM`, `_TIMESTAMPTZ`, `_TIMESTAMPLTZ`, `_DATETIMETZ`, `_DATETIMELTZ`, `_TIMETZ`, `_JSON`.

### Key structs
- **`T_CCI_ERROR`** = `{char err_msg[1024]; int err_code;}` — fixed 1024-byte ABI commitment.
- **`T_CCI_BIT`** = `{int size; char *buf;}`
- **`T_CCI_DATE`** = `{short yr,mon,day,hh,mm,ss,ms;}`
- **`T_CCI_DATE_TZ`** adds `char tz[64]` to T_CCI_DATE
- **`T_CCI_COL_INFO`** — column metadata (type, precision, scale, nullable, name, default, primary key, unique, reverse, foreign key, shared, class name, attr name)
- **`T_CCI_QUERY_RESULT`**, `T_CCI_PARAM_INFO`

### Enums
- `T_CCI_DB_PARAM` — `CCI_PARAM_ISOLATION_LEVEL`, `CCI_PARAM_LOCK_TIMEOUT`, `CCI_PARAM_MAX_STRING_LENGTH`, `CCI_PARAM_AUTO_COMMIT`
- `T_CCI_SCH_TYPE` — `CCI_SCH_*` schema info types (~17 values)
- `T_CCI_CUBRID_STMT` — statement type returned by `cci_get_result_info`
- `T_CCI_CURSOR_POS` — `CCI_CURSOR_FIRST`, `CCI_CURSOR_CURRENT`, `CCI_CURSOR_LAST`
- `T_CCI_TRAN_ISOLATION` — `TRAN_READ_COMMITTED=4`, `TRAN_REP_READ=5`, `TRAN_SERIALIZABLE=6`
- `T_CCI_PARAM_MODE` — `CCI_PARAM_MODE_IN`, `OUT`, `INOUT`

### `cci_execute` flags
- `CCI_EXEC_QUERY_ALL` — execute all `;`-separated queries; first result returned, others via `cci_next_result`. **No rollback** of prior queries on mid-batch failure.
- `CCI_EXEC_ASYNC` — **removed** in 2008 R4.4 / 9.2.

## Error Codes

- **CCI errors**: `-20001..-20999`. `cci.rst:642-647`.
- `-20001 CCI_ER_DBMS` = server returned an error. Real server `err_code` lives in `T_CCI_ERROR.err_code`.
- `-20036 CCI_ER_DATASOURCE_TIMEOUT` — pool exhausted.
- **CAS errors transferred via CCI**: `-10001..-10999`. `cci.rst:646`.

## Programming Model (cci.rst overview)

The standard sequence:
```
cci_connect → cci_prepare → cci_bind_param → cci_execute → cci_cursor + cci_fetch + cci_get_data → cci_close_req_handle → cci_disconnect
```

Pool variant:
```
cci_property_create → cci_datasource_create → cci_datasource_borrow → ... → cci_datasource_release → cci_datasource_destroy
```

## Threading & Autocommit

- **Per-connection only** — connection handles are NOT thread-safe. One connection per thread.
- **Autocommit + SELECT**: in autocommit mode, txn is NOT committed until ALL results are fetched. If error during fetch, must call `cci_end_tran`.
- Default autocommit: inherits broker `CCI_DEFAULT_AUTOCOMMIT` (default ON since 4.0+).

## SSL

- `useSSL=true` URL property opt-in.
- Server enables via per-broker `SSL=ON` in `cubrid_broker.conf`.
- Mismatch = connection rejection (no negotiation).

## Cross-References

- [[modules/cubrid-cci]] — submodule containing CCI implementation (pending source ingest)
- [[components/cas]] — broker terminus that CCI talks to
- [[components/broker-impl]] — `CCI_DEFAULT_AUTOCOMMIT` inheritance
- [[components/connection]] — wire protocol, `altHosts` failover
- [[components/dbi-compat]] — DB_VALUE ↔ T_CCI_* marshalling
- [[sources/cubrid-manual-jdbc]] — JDBC, the parallel surface in Java
- [[sources/cubrid-manual-error-codes]] — full error code catalogue

## Incidental Wiki Enhancements

- [[components/cas]]: documented the CCI URL format (`cci:CUBRID:<host>:<port>:<db>:<user>:<pw>:[?<props>]`) and the case-insensitive snake_case ↔ camelCase property aliases.
- [[components/connection]]: documented `disconnectOnQueryTimeout` semantics (false = wait for server response on timeout, true = close socket immediately and return CCI_ER_QUERY_TIMEOUT; broker may continue executing the query).
- [[components/dbi-compat]]: documented the A-type vs U-type duality and the 17 `CCI_SCH_*` schema constants.

## Key Insight

CCI is **the wire-protocol contract** in C form. Its 95 functions cover everything the wire supports; everything else (JDBC, PHP, Python, ...) is a thin or thick wrapper. The **A-type / U-type duality** is unique to CCI — A-types describe how the C application stores the value, U-types describe how the DB stores it; the bind layer converts. The `T_CCI_ERROR` 1024-byte fixed-buffer ABI is a real commitment that downstream wrappers depend on.
