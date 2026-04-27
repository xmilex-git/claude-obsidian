---
type: stub
title: "cubrid-cci (submodule)"
created: 2026-04-27
updated: 2026-04-27
tags:
  - module
  - cubrid
  - cci
  - submodule
  - stub
status: stub
related:
  - "[[modules/_index]]"
  - "[[sources/cubrid-manual-cci]]"
  - "[[components/cas]]"
  - "[[components/dbi-compat]]"
  - "[[Architecture Overview]]"
  - "[[Tech Stack]]"
  - "[[hot]]"
---

# cubrid-cci (Submodule)

**Status:** Stub. Pending dedicated source ingest.

## What It Is

CUBRID Call Interface — the **C-language driver** for CUBRID. Distributed as a separate Git submodule from the main `~/dev/cubrid/` repository because it's the foundation other drivers (PHP, PDO, Perl, Python, Ruby, ODBC) wrap.

## Where It Lives

- **Submodule path** (when checked out): `~/dev/cubrid/cubrid-cci/` (or as referenced by the main repo's `.gitmodules`)
- **Upstream**: `https://github.com/CUBRID/cubrid-cci`
- **Installed artifacts**: `$CUBRID/cci/include/cas_cci.h`, `$CUBRID/cci/lib/libcascci.{so,a}`, `$CUBRID/cci/bin/cascci.dll` (Windows)

## What It Does

- Implements the wire protocol for talking to a `cub_cas` (CAS) worker.
- Provides ~95 `cci_*` C functions covering connect, prepare, execute, fetch, transaction, blob/clob, schema introspection, and the connection pool (Datasource).
- Defines the public ABI: `T_CCI_ERROR`, `T_CCI_DATE`, `T_CCI_DATE_TZ`, `T_CCI_BIT`, `T_CCI_COL_INFO`, `T_CCI_QUERY_RESULT`, `T_CCI_PARAM_INFO`, plus `T_CCI_U_TYPE` / `T_CCI_A_TYPE` parallel type universes.
- Owns connection-string parsing: `cci:CUBRID:<host>:<port>:<db>:<user>:<pw>:[?<props>]`.
- Owns HA failover client-side: `altHosts`, `loadBalance`, `rcTime` URL properties.

## Documentation

- **End-user reference**: [[sources/cubrid-manual-cci]] catalogs all 95 functions, type system, error codes, programming model.
- **Implementation source**: pending dedicated ingest. When done, a `wiki/sources/cubrid-cci-src.md` will document the internal modules.

## Why a Separate Submodule

CCI predates the modern CUBRID source layout and was designed to be linkable independently (so light-touch language drivers could ship without compiling all of CUBRID). Keeping it as a separate Git repo means JDBC, PHP, etc. can pin to a specific CCI version without dragging in the whole engine.

## Related

- [[sources/cubrid-manual-cci]] — manual reference (95 functions catalogued)
- [[sources/cubrid-manual-api]] — driver overview (CCI is the foundation for most)
- [[components/cas]] — the broker terminus CCI talks to
- [[components/broker-impl]] — `CCI_DEFAULT_AUTOCOMMIT` and other CAS-side defaults inherited by CCI
- [[components/dbi-compat]] — DB_VALUE ↔ T_CCI_* marshalling on the engine side
- [[modules/_index]]
- Open follow-up: dedicated source ingest (flagged in [[hot]])
