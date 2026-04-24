---
type: module
path: "broker/"
status: active
language: "C/C++17"
purpose: "Top-level CMake target for the CUBRID broker; configs and SSL certs"
last_updated: 2026-04-23
depends_on:
  - "[[components/broker-impl|src/broker]]"
used_by: []
tags:
  - module
  - cubrid
  - broker
related:
  - "[[modules/src|src module]]"
  - "[[components/broker-impl|src/broker (impl)]]"
  - "[[Architecture Overview]]"
  - "[[Data Flow]]"
created: 2026-04-23
updated: 2026-04-23
---

# `broker/` — Broker CMake Target

Top-level directory for the **broker target**: configs, SSL certs, and the CMake glue that builds the broker binary.

> [!warning] Not the implementation
> The actual broker source code lives in [[components/broker-impl|src/broker/]]. This top-level `broker/` directory only assembles configs and the build target. Don't edit broker behavior here.

## What it contains

- Broker configuration templates (e.g., `cubrid_broker.conf` skeleton)
- SSL certificate scaffolding for secure broker connections
- CMake target definition

## Role in the topology

```
Client ──► Broker (this) ──► CAS workers ──► DB Server
```

Broker is a multi-process connection router: it listens for client connections, pools CAS (CUBRID Application Server) worker processes, and dispatches each connection to a worker. CAS then talks to the DB server.

## Configs of interest

(Will be filled when [[modules/conf|conf/]] is ingested.)

## Related

- Implementation: [[components/broker-impl|src/broker]]
- CAS lifecycle: TBD (separate ingest)
- Source guide: [[cubrid-AGENTS]]
