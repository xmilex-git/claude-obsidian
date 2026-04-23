---
type: component
parent_module: "[[modules/src|src]]"
path: "src/api/"
status: active
purpose: "Public C API extensions beyond the standard db_* family in compat. Currently the sole resident is the CDC (Change Data Capture) interface built on log supplemental information."
key_files:
  - "cubrid_log.h — CDC public header: CUBRID_LOG_ITEM, DDL/DML/DCL/TIMER types, four-phase API"
  - "cubrid_log.c — CDC client implementation: connect, find-LSA, extract, clear, finalize"
tags:
  - component
  - cubrid
  - api
  - cdc
  - change-data-capture
related:
  - "[[components/compat|compat]]"
  - "[[components/client-api|client-api]]"
  - "[[components/cubrid-log-cdc|cubrid-log-cdc]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/connection|connection]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/api/` — Public C API Extensions

`src/api/` holds public C interfaces that extend CUBRID's application-facing surface beyond the `db_*` family in [[components/compat|compat]]. Unlike `src/compat/`, which covers generic DML/DDL/connection APIs, `src/api/` delivers purpose-built, higher-level interfaces intended for external tools and integrations.

## Current Residents

| File | Component | Role |
|------|-----------|------|
| `cubrid_log.h` | [[components/cubrid-log-cdc\|cubrid-log-cdc]] | CDC public header |
| `cubrid_log.c` | [[components/cubrid-log-cdc\|cubrid-log-cdc]] | CDC client implementation |

> [!key-insight] CS_MODE only
> The entire `cubrid_log.c` is wrapped in `#if defined(CS_MODE)`. The CDC client is a pure client-library feature — it opens a dedicated raw CSS connection to the server and speaks the CDC sub-protocol. It cannot run in `SA_MODE` or `SERVER_MODE`.

## Design Intent

Files in `src/api/` are intended for **non-broker** consumption — direct application programs that want raw log event streams, administrative tooling, or other capabilities orthogonal to SQL execution. They may depend on the connection (`src/connection/`) and object representation (`object_representation.h`) layers, but avoid the full broker–CAS stack.

## Related

- [[components/compat|compat]] — `db_*` standard application API
- [[components/client-api|client-api]] — detailed `db_*` function families
- [[components/cubrid-log-cdc|cubrid-log-cdc]] — CDC sub-component detail
- [[components/log-manager|log-manager]] — server WAL that CDC reads
- [[Build Modes (SERVER SA CS)]] — why CS_MODE guard matters
- Source: [[sources/cubrid-src-api|cubrid-src-api]]
