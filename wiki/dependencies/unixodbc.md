---
type: dependency
name: "unixODBC"
version: "2.3.9"
source: "https://github.com/CUBRID/3rdparty/raw/develop/unixODBC/unixODBC-2.3.9.tar.gz"
license: "LGPL-2.1"
bundled: true
used_by:
  - "ODBC driver manager — enables CUBRID ODBC driver connectivity"
risk: low
tags:
  - dependency
  - cubrid
  - odbc
  - connectivity
created: 2026-04-23
updated: 2026-04-23
---

# unixODBC

## What it does

unixODBC is the standard ODBC driver manager for UNIX/Linux systems. It provides the ODBC API (`libodbc`) that ODBC-based applications link against; the driver manager then loads the appropriate CUBRID ODBC driver at runtime.

## Why CUBRID uses it

CUBRID ships an ODBC driver (`cubrid-odbc`). On Linux, the ODBC driver manager required is unixODBC. CUBRID bundles unixODBC so it controls the exact version and avoids dependency on whatever version the host system has installed.

## Integration points

- CMake target: `libodbc`
- Linux only: built from source via `ExternalProject_Add`
- **Shared library** (`libodbc.so`) — unlike other 3rdparty libs which build static, unixODBC is built as a shared library (no `--enable-static` flag used)
- Configure: `<SOURCE_DIR>/configure --prefix=${3RDPARTY_LIBS_DIR}` (without `--enable-static --disable-shared`)
- Only `EXTERNAL` mode is supported; `SYSTEM` mode raises a `FATAL_ERROR`
- Exposes via `expose_3rdparty_variable(LIBUNIXODBC)`

## Risk / notes

- LGPL-2.1 — shared-library (dynamic linking) model is compatible with CUBRID's Apache 2.0 license.
- v2.3.9 (2019) is recent for the 2.3.x series; upstream 2.3.x is stable.
- Only `EXTERNAL` mode is allowed — cannot be overridden to use a system unixODBC. This is intentional but means CUBRID always builds its own copy.

## Related

- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
