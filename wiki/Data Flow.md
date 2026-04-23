---
type: overview
title: "CUBRID Data Flow"
updated: 2026-04-23
tags:
  - cubrid
  - flow
  - overview
status: stub
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

## Other flows to document

- Commit + log write + recovery
- HA replication (primary → standby)
- Prepared statement / cursor lifecycle
- Stored procedure dispatch ([[pl_engine]])

Each gets its own page under [[flows/_index|flows/]] as ingest runs.
