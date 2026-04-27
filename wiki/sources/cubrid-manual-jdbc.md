---
created: 2026-04-27
type: source
title: "CUBRID Manual — JDBC Driver (api/jdbc.rst)"
source_path: "/home/cubrid/cubrid-manual/en/api/jdbc.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - jdbc
  - java
  - driver
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-cci]]"
  - "[[sources/cubrid-manual-api]]"
  - "[[modules/cubrid-jdbc]]"
  - "[[components/cas]]"
  - "[[components/cursor]]"
  - "[[components/connection]]"
---

# CUBRID Manual — JDBC Driver (api/jdbc.rst)

**Ingested:** 2026-04-27
**Source file:** `/home/cubrid/cubrid-manual/en/api/jdbc.rst` (1603 lines)
**Implementation:** lives in submodule `modules/cubrid-jdbc` (pending dedicated source ingest)

## What This Covers

The **JDBC driver reference** for CUBRID. Built against JDBC 2.0 spec, compiled for JDK 1.8 (with one JDBC 3.0 method supported: `Statement.getGeneratedKeys()`). Reimplements the wire protocol directly (does NOT wrap CCI like the other drivers).

## Section Map

| Section | Content |
|---|---|
| **Overview** | `cubrid_jdbc.jar` location/version/install, CLASSPATH setup |
| **Class.forName** | `cubrid.jdbc.driver.CUBRIDDriver` |
| **getConnection** | URL grammar + ALL property reference |
| **CUBRIDDataSource / CUBRIDConnectionPoolDataSource** | Pool helpers |
| **Foreign-key metadata** | DatabaseMetaData methods supported |
| **OIDs and Collections** | `CUBRIDResultSet` extensions, `cubrid.sql.*` package |
| **Auto-increment** | `getGeneratedKeys()` |
| **BLOB/CLOB** | LOB programming model |
| **JDBC error code table** | -21001..-21141 catalogue |
| **JDBC standard/extended interface support matrix** | What is and isn't supported |
| **Sample programs** | End-to-end examples |

## URL Grammar

```
jdbc:cubrid:<host>:<port>:<db>:[user]:[pw]:[?<prop>[&<prop>]]
```

### Properties (15+)
- `altHosts` — comma-separated fallback brokers (HA failover)
- `loadBalance` — `false` / `true|rr` (round-robin) / `sh` (shuffle)
- `rcTime` — reconnect-to-primary interval after failover
- `connectTimeout` — initial connect timeout per host
- `queryTimeout` — per-query timeout (whole `executeBatch()` honors this)
- `charSet` — driver-side charset
- `zeroDateTimeBehavior` — how to handle '0000-00-00' values
- `logFile`, `logOnException`, `logSlowQueries`, `slowQueryThresholdMillis`
- `useLazyConnection` — defer actual TCP connect
- `useSSL` — TLS/SSL opt-in (must match broker `SSL=ON`)
- `clientCacheSize` — fetch buffer size
- `usePreparedStmtCache`, `preparedStmtCacheSize`, `preparedStmtCacheSqlLimit`
- `hold_cursor` — defaults `true` since 9.1 (HOLD_CURSORS_OVER_COMMIT)

## Default Port

**33000** (matches CCI). ADO.NET is the only outlier at 30000.

## CUBRID-specific Extensions

Package `cubrid.sql`:
- **`CUBRIDOID`** — opaque object identifier; useful for OID-based access (rare)
- **`CUBRIDDataSource`** — DataSource for connection pooling
- **`CUBRIDConnectionPoolDataSource`** — pooled DataSource

Class `cubrid.jdbc.driver.*`:
- **`CUBRIDConnection`** — extends Connection
- **`CUBRIDStatement`**, **`CUBRIDPreparedStatement`** — extends standard
- **`CUBRIDResultSet`** — adds OID/collection accessors, BLOB/CLOB methods
- **`CUBRIDResultSetMetaData`**
- **`CUBRIDException`**

## Standard JDBC Support Matrix

### Supported
- Standard JDBC 2.0 surface
- `Statement.getGeneratedKeys()` (JDBC 3.0) — only this one from JDBC 3.0+

### NOT supported (must avoid in portable Java code)
- `java.sql.Array`
- `java.sql.ParameterMetaData`
- `java.sql.Ref`
- `java.sql.Savepoint`
- `java.sql.SQLData`
- `java.sql.SQLInput`
- `java.sql.Struct`

`jdbc.rst:1591-1597` enumerates this.

## Cursor Holdability (`jdbc.rst:213, 1411`)

- `hold_cursor` URL property defaults to `true` since CUBRID 9.1.
- Means: `ResultSet` survives transaction commit (HOLD_CURSORS_OVER_COMMIT).
- Pre-9.1: cursor closed at commit.

## Threading

Per-connection. **Connection objects are NOT thread-safe.** Standard JDBC contract. Same statement repeated across drivers: *"The database connection in thread-based programming must be used independently each other."*

## Autocommit

- Default ON (inherits CCI default since 4.0+).
- **Extended-API `CUBRIDStatement` does NOT autocommit even when AUTOCOMMIT=ON** (`jdbc.rst:604-606`). This is a documented pitfall — using the extended interface forces explicit `commit()`/`rollback()`.

## executeBatch semantics changed in 4.3

- **Pre-4.3**: `executeBatch()` committed once at the end of the array in autocommit mode.
- **From 4.3**: `executeBatch()` commits per-row in autocommit mode.
- Same change applied to CCI `cci_execute_array`.

## queryTimeout for executeBatch

- Applies to the whole `executeBatch()` call, not per-query (`jdbc.rst:181`).
- Same caveat for CCI `cci_execute_batch` and `CCI_EXEC_QUERY_ALL`.

## Error Code Range

**-21001..-21999** for JDBC-side errors.
- `-21001..-21024` — protocol/data-mapping errors
- `-21101..-21141` — JDBC API misuse (closed connection, unsupported method, etc.)

Server errors arrive as standard `SQLException` chain via `CCI_ER_DBMS` translation.

## SSL

- `useSSL=true` URL property opt-in.
- Server enables per-broker via `SSL=ON` in `cubrid_broker.conf`.
- Mismatch = connection rejection at broker.

## HA Failover (altHosts)

- `altHosts=h2,h3,h4` — driver-side failover list.
- `loadBalance=true|rr|sh` — initial selection strategy.
- `rcTime` — when to retry primary after failover.
- **ACCESS_MODE on alternate brokers is IGNORED** by client-side selection — driver picks regardless of RW/RO/SO mode (`jdbc.rst:164-175`). Application must handle ReadOnly errors.

## Sample (canonical)

```java
Class.forName("cubrid.jdbc.driver.CUBRIDDriver");
Connection conn = DriverManager.getConnection(
    "jdbc:cubrid:localhost:33000:demodb:::?charSet=utf-8&queryTimeout=5000",
    "dba", "");
PreparedStatement ps = conn.prepareStatement("SELECT name FROM athlete WHERE nation_code=?");
ps.setString(1, "KOR");
ResultSet rs = ps.executeQuery();
while (rs.next()) {
    System.out.println(rs.getString("name"));
}
rs.close();
ps.close();
conn.close();
```

## Cross-References

- [[modules/cubrid-jdbc]] — submodule with JDBC implementation (pending source ingest)
- [[sources/cubrid-manual-cci]] — companion C API; many semantics shared
- [[components/cas]] — wire-protocol terminus
- [[components/cursor]] — server-side cursor + holdability
- [[components/connection]] — `altHosts`, HA failover

## Incidental Wiki Enhancements

- [[components/cursor]]: documented JDBC `hold_cursor=true` URL default since 9.1 (HOLD_CURSORS_OVER_COMMIT) — server-side cursor survives commit by default.
- [[components/connection]]: documented JDBC `altHosts`/`loadBalance`/`rcTime` URL grammar, the `false`/`rr`/`sh` selection modes, and the documented behavior that ACCESS_MODE on alternate brokers is IGNORED by client-side selection.
- [[components/client-api]]: documented JDBC unsupported standard interfaces (Array, ParameterMetaData, Ref, Savepoint, SQLData, SQLInput, Struct) — only `getGeneratedKeys()` from JDBC 3.0 supported on Statement.

## Key Insight

JDBC is **NOT a CCI wrapper** — it reimplements the wire protocol in pure Java. This makes JDBC and CCI parallel (not stacked), which is why `executeBatch()` semantics had to change in both at the same version. **Two operational gotchas**: (1) Extended-API `CUBRIDStatement` doesn't autocommit even when AUTOCOMMIT=ON — silent correctness pitfall; (2) ACCESS_MODE is ignored by `altHosts` selection — client must handle ReadOnly errors after failover.
