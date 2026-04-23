---
type: overview
title: "CUBRID Data Flow"
updated: 2026-04-23
tags:
  - cubrid
  - flow
  - overview
status: developing
related:
  - "[[Architecture Overview]]"
  - "[[flows/_index|Flows]]"
---

# CUBRID Data Flow

> Stub. Expanded by ingest of `broker/`, `cs/`, and `src/` query path code.

## Query path (high level)

```
Client (CCI / JDBC)
    │
    ▼
Broker (broker/) ── connection pool / router
    │
    ▼
CAS (CUBRID Application Server worker)
    │
    ▼
DB Server (src/)
    ├── Parser
    ├── Optimizer
    ├── Executor
    ├── Transaction / Log
    └── Storage (page buffer, B+tree, heap)
```

## Detailed query path

See [[Query Processing Pipeline]] for the full lexer → parser → name resolution → semantic check → XASL gen → serialize → deserialize → execute trace, with file/function pointers.

## Other flows to document

- Commit + log write + recovery — files in [[components/transaction|`src/transaction/log_*`]]
- HA replication (primary → standby)
- Prepared statement / cursor lifecycle
- Stored procedure dispatch — [[components/sp]] + [[modules/pl_engine|pl_engine]]
- LOB read/write (cross-cutting: [[components/object|object/lob_locator.cpp]] + [[components/storage|storage/es.c]])

Each gets its own page under [[flows/_index|flows/]] as ingest runs.
