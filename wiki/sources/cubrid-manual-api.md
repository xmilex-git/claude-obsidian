---
created: 2026-04-27
type: source
title: "CUBRID Manual — API Drivers (api/, excluding CCI and JDBC)"
source_path: "/home/cubrid/cubrid-manual/en/api/index.rst, php.rst, pdo.rst, odbc.rst, adodotnet.rst, perl.rst, python.rst, ruby.rst, node_js.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - api
  - drivers
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-cci]]"
  - "[[sources/cubrid-manual-jdbc]]"
  - "[[components/cas]]"
  - "[[components/broker-impl]]"
  - "[[components/dbi-compat]]"
---

# CUBRID Manual — API Drivers (excluding CCI and JDBC)

**Ingested:** 2026-04-27
**Source files:** `api/index.rst` (26), `php.rst` (672), `pdo.rst` (537), `odbc.rst` (464), `adodotnet.rst` (507), `perl.rst` (70), `python.rst` (248), `ruby.rst` (260), `node_js.rst` (48). Total ~2902 lines.
**Companion pages:** [[sources/cubrid-manual-cci]] for the C API; [[sources/cubrid-manual-jdbc]] for Java.

## Driver Inventory & Architecture Map

| Driver | Lines | Backed by | Connection string default port | Autocommit default |
|---|---|---|---|---|
| CCI (C) | 5099 | — (foundation) | 33000 | inherits broker `CCI_DEFAULT_AUTOCOMMIT` (default ON) |
| JDBC (Java) | 1603 | own wire reimpl | 33000 | inherits CCI default (ON) |
| **PHP (`cubrid` PECL)** | 672 | CCI wrapper | 33000 | inherits CCI default |
| **PDO (`pdo_cubrid` PECL)** | 537 | CCI wrapper | 33000 | inherits CCI default |
| **ODBC** | 464 | CCI-based | 33000 | **explicitly ignores `CCI_DEFAULT_AUTOCOMMIT` since 9.3** — own AUTOCOMMIT property |
| **ADO.NET** | 507 | own managed reimpl | **30000** (the only outlier) | own — explicit in connection string |
| **Perl (DBD::cubrid)** | 70 | DBI / CCI | 33000 | inherits CCI default |
| **Python (`CUBRIDdb` + `_cubrid`)** | 248 | DB-API 2.0 over CCI | 33000 | **OFF by default in this driver** (overrides CCI default) |
| **Ruby (cubrid gem)** | 260 | CCI wrapper | 33000 | inherits CCI default |
| **Node.js (node-cubrid)** | 48 | own pure-JS reimpl | 33000 | not enumerated |

**Two non-CCI drivers**: ADO.NET (managed C# in `github.com/CUBRID/cubrid-adonet`) and Node.js (pure JS in `github.com/CUBRID/node-cubrid`). Everything else is CCI-based.

## Connection String Catalogue

```
CCI:        cci:CUBRID:<host>:<port>:<db>:<user>:<pw>:[?<props>]
JDBC:       jdbc:cubrid:<host>:<port>:<db>:[user]:[pw]:[?<prop>[&<prop>]]
ODBC:       DRIVER={CUBRID Driver};SERVER=...;PORT=...;UID=...;PWD=...;DB_NAME=...;CHARSET=...;AUTOCOMMIT=...;OMIT_SCHEMA=...;FETCH_SIZE=...
ADO.NET:    server=...;database=...;port=...;user=...;password=...;
PDO:        cubrid:host=127.0.0.1;port=33000;dbname=demodb
```

## Per-Driver Highlights

### PHP (`cubrid` PECL extension, `cubrid.so` / `php_cubrid.dll`)
- Functions all prefixed `cubrid_*` (`cubrid_connect`, `cubrid_execute`, `cubrid_fetch`, `cubrid_set_autocommit`, `cubrid_schema`, `cubrid_set_add`, `cubrid_set_drop`).
- Schema lookup constants `CUBRID_SCH_*`.
- OID + collection helpers — exposes CUBRID-specific OIDs and SET/MULTISET/LIST.
- Function reference is **off-site** (php.net).
- Install via PECL, apt-get, or Windows installer.

### PDO (PECL `pdo_cubrid`)
- DSN: `cubrid:host=...;port=...;dbname=...`
- 17 `PDO::CUBRID_SCH_*` constants for schema metadata: TABLE, VIEW, QUERY_SPEC, ATTRIBUTE, TABLE_ATTRIBUTE, TABLE_METHOD, METHOD_FILE, SUPER_TABLE, SUB_TABLE, CONSTRAINT, TRIGGER, TABLE_PRIVILEGE, COL_PRIVILEGE, DIRECT_SUPER_TABLE, DIRECT_PRIMARY_KEY, IMPORTED_KEYS, EXPORTED_KEYS, CROSS_REFERENCE.
- `PDO::cubrid_schema()` is the CUBRID-specific extension method.
- BLOB/CLOB support.
- Honors `CCI_DEFAULT_AUTOCOMMIT` since it's a CCI wrapper.

### ODBC (3.52)
- Three deviations from CCI: (a) ignores `CCI_DEFAULT_AUTOCOMMIT` since 9.3 (own AUTOCOMMIT property), (b) supports `OMIT_SCHEMA=YES` for ERwin compat on 11.2+, (c) ships separate `{CUBRID Driver Unicode}` since 9.3.0.0002.
- Full DSN configuration via Driver Manager OR DSN-less `driver={CUBRID Driver}` connection string.
- ODBC type map is the only canonical CUBRID↔ODBC translation table.
- Supports ASP/VBScript via DSN.

### ADO.NET (the architectural outlier)
- Pure managed C# — `Cubrid.Data.dll` from `github.com/CUBRID/cubrid-adonet`.
- **No CUBRID install required on the client machine** (only the .NET assembly).
- Default port **30000**, not 33000.
- Standard ADO.NET surface: `CUBRIDConnection`, `CUBRIDCommand`, `CUBRIDDataReader`, `CUBRIDConnectionStringBuilder`, `CUBRIDOid`, `CUBRIDBlob`, `CUBRIDClob`, `CUBRIDSchemaProvider`.
- Batch helpers: `BatchExecute`, `BatchExecuteNoQuery`.

### Perl (DBD::cubrid)
- DBI-compliant. Install via CPAN or source. Requires CCI driver.
- LOB and column metadata not supported.
- API hosted off-site.

### Python (`CUBRIDdb` over `_cubrid`)
- DB-API 2.0 wrapper around lower-level `_cubrid` C extension.
- `paramstyle='qmark'`, `threadsafety=2`, `apilevel='2.0'`.
- `connect('CUBRID:host:port:db:::', user, pw)` connection format.
- **Auto-commit defaults OFF** in this driver (overrides CCI default — Python is the exception).

### Ruby (cubrid gem + ActiveRecord adapter)
- `gem install cubrid`, `adapter => "cubrid"` in ActiveRecord config.
- Supported `:type` for migrations: `:string :text :integer :float :decimal :datetime :timestamp :time :boolean :bit :smallint :bigint :char`.
- **`:binary` not supported.**
- `create_database` not supported programmatically.
- Inherits `CCI_DEFAULT_AUTOCOMMIT`.

### Node.js (node-cubrid)
- 100% JavaScript, no native compilation.
- `npm install node-cubrid`.
- Requires CUBRID 8.4.1 P2 or later.
- Source: `github.com/CUBRID/node-cubrid`.
- API hosted off-site.

## Cross-Driver Invariants

- **Per-connection threading**: every driver's docs include the line *"The database connection in thread-based programming must be used independently each other."* Repeated verbatim across cci.rst:66, jdbc.rst:277, odbc.rst:159, perl.rst:17, php.rst:478, adodotnet.rst:164.
- **Autocommit + SELECT contract**: *"In autocommit mode, the transaction is not committed if all results are not fetched after running SELECT … you should end the transaction by calling cci_end_tran (or COMMIT/ROLLBACK) if some error occurs during fetching."* Identical wording in cci.rst:67, jdbc.rst:279, php.rst:481, odbc.rst:160, perl.rst:18.
- **Error code namespaces**: see [[sources/cubrid-manual-error-codes]] for the full -10K (CAS) / -20K (CCI) / -21K (JDBC) split.

## Cross-References

- [[sources/cubrid-manual-cci]] — CCI driver (foundation)
- [[sources/cubrid-manual-jdbc]] — JDBC driver
- [[components/cas]] — CAS terminates every CCI/JDBC/ODBC/PHP/PDO/Perl/Python/Ruby connection
- [[components/broker-impl]] — broker port matrix, `CCI_DEFAULT_AUTOCOMMIT`
- [[components/dbi-compat]] — driver-side DB_VALUE marshalling

## Incidental Wiki Enhancements

- [[components/cas]]: documented the 5 connection-string syntax variants (CCI / JDBC / ODBC / ADO.NET / PDO) and cas.md noted as the entry point each driver's `fn_*` handlers terminate.
- [[components/broker-impl]]: documented the 33000-vs-30000 default-port split (everyone except ADO.NET uses 33000; ADO.NET uses 30000) and `CCI_DEFAULT_AUTOCOMMIT` inheritance map (PHP/PDO/Perl/Ruby inherit; Python overrides OFF; ODBC ignores since 9.3).
- [[components/dbi-compat]]: documented error-code namespace partitioning (CCI -20001..-20999, CAS -10001..-10200, JDBC -21001..-21999, server below -9999); `T_CCI_ERROR` is `{char err_msg[1024]; int err_code;}` — the 1024-byte fixed buffer is a public ABI commitment.

## Key Insight

**Eight of CUBRID's nine drivers are CCI wrappers.** The two exceptions — ADO.NET (.NET managed) and Node.js (pure JS) — reimplement the wire protocol. This makes CCI the de facto wire-protocol spec; drivers like PHP/PDO/Perl/Ruby/Python/ODBC inherit defaults and behavior from CCI. **Three subtle gotchas**: (1) ADO.NET uses port 30000 not 33000; (2) Python overrides autocommit to OFF (sole exception in CCI family); (3) ODBC ignores `CCI_DEFAULT_AUTOCOMMIT` (sole exception in CCI family) and uses its own AUTOCOMMIT property.
