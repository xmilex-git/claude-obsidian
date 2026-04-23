---
type: index
title: "CUBRID Flows"
updated: 2026-04-23
tags:
  - index
  - cubrid
  - flow
status: active
---

# Flows

Request paths, data flows, auth flows, and lifecycle sequences. Examples:

- Client → Broker → CAS → DB Server query path
- Transaction commit + log recovery flow
- HA replication flow
- Parser → optimizer → executor pipeline
- Lock acquisition and deadlock detection

## Pages

| Page | Entry point | Exit point |
|------|-------------|------------|
| [[flows/dml-execution-path\|dml-execution-path]] | INSERT/UPDATE/DELETE/MERGE SQL | Modified rows + commit ack |
| [[flows/ddl-execution-path\|ddl-execution-path]] | CREATE/ALTER/DROP/GRANT/REVOKE SQL | Schema commit + catalog updated |

Populated by `ingest`.

Navigation: [[index]] | [[modules/_index|Modules]] | [[Data Flow]]
