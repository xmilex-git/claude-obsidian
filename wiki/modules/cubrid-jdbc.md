---
type: stub
title: "cubrid-jdbc (submodule)"
created: 2026-04-27
updated: 2026-04-27
tags:
  - module
  - cubrid
  - jdbc
  - submodule
  - java
  - stub
status: stub
related:
  - "[[modules/_index]]"
  - "[[sources/cubrid-manual-jdbc]]"
  - "[[components/cas]]"
  - "[[components/cursor]]"
  - "[[components/connection]]"
  - "[[Architecture Overview]]"
  - "[[Tech Stack]]"
  - "[[hot]]"
---

# cubrid-jdbc (Submodule)

**Status:** Stub. Pending dedicated source ingest.

## What It Is

The **JDBC driver** for CUBRID — pure-Java, reimplements the wire protocol directly (does NOT wrap CCI). Built against the JDBC 2.0 spec, compiled for JDK 1.8, with one JDBC 3.0 method supported (`Statement.getGeneratedKeys()`).

## Where It Lives

- **Submodule path** (when checked out): `~/dev/cubrid/cubrid-jdbc/` (or as referenced by the main repo's `.gitmodules`)
- **Upstream**: `https://github.com/CUBRID/cubrid-jdbc`
- **Installed artifact**: `$CUBRID/jdbc/cubrid_jdbc.jar`
- **Driver class**: `cubrid.jdbc.driver.CUBRIDDriver`

## What It Does

- Implements the wire protocol in pure Java — no JNI, no native bridge.
- URL grammar: `jdbc:cubrid:<host>:<port>:<db>:[user]:[pw]:[?<prop>[&<prop>]]`
- Owns the `cubrid.sql.*` extension package: `CUBRIDOID`, `CUBRIDDataSource`, `CUBRIDConnectionPoolDataSource`.
- Owns the `cubrid.jdbc.driver.*` extended classes: `CUBRIDConnection`, `CUBRIDStatement`, `CUBRIDPreparedStatement`, `CUBRIDResultSet`, `CUBRIDResultSetMetaData`, `CUBRIDException`.
- HA failover via `altHosts`, `loadBalance` (`true`/`rr`/`sh`), `rcTime`.

## Why a Separate Submodule

JDBC has its own release cadence and a Java toolchain (Maven + JDK), distinct from the C/C++ engine build. Keeping it separate avoids forcing engine builders to install Java, and avoids forcing Java consumers to compile C++.

## Important Notes

- **NOT a CCI wrapper** — parallel implementation. JDBC and CCI versions are released together but neither depends on the other's binary.
- Extended-API `CUBRIDStatement` does NOT autocommit even when AUTOCOMMIT=ON — silent correctness pitfall.
- ACCESS_MODE on `altHosts` brokers is IGNORED by client-side selection — driver picks regardless of RW/RO/SO mode.
- `hold_cursor=true` URL property defaults to `true` since CUBRID 9.1 (HOLD_CURSORS_OVER_COMMIT).

## Documentation

- **End-user reference**: [[sources/cubrid-manual-jdbc]] catalogs URL grammar, properties, support matrix, error codes (-21001..-21999), and the extension API.
- **Implementation source**: pending dedicated ingest.

## Related

- [[sources/cubrid-manual-jdbc]] — manual reference
- [[sources/cubrid-manual-cci]] — companion C API; many semantics shared (autocommit + SELECT contract, etc.)
- [[components/cas]] — wire-protocol terminus
- [[components/cursor]] — server-side cursor + holdability
- [[components/connection]] — `altHosts`, HA failover
- [[components/client-api]] — driver-side API surface
- [[modules/_index]]
- Open follow-up: dedicated source ingest (flagged in [[hot]])
