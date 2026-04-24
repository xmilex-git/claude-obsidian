---
created: 2026-04-23
type: overview
title: "CUBRID Architecture Overview"
updated: 2026-04-23
tags:
  - cubrid
  - architecture
  - overview
status: developing
related:
  - "[[modules/_index|Modules]]"
  - "[[Tech Stack]]"
  - "[[Data Flow]]"
  - "[[Dependency Graph]]"
  - "[[Key Decisions]]"
---

# CUBRID Architecture Overview

## At a glance

CUBRID is an open-source, ACID-compliant relational database management system written primarily in C/C++17 with a Java PL engine for stored procedures. Source tree: `/Users/song/DEV/cubrid/`. Apache 2.0. v11.5.x.

Three-process-group topology:

1. **Client / driver** ([[modules/cubrid-cci|cubrid-cci]], [[modules/cubrid-jdbc|cubrid-jdbc]], client libs in [[modules/cs|cs/]])
2. **Broker** ([[modules/broker|broker/]]) — multi-process connection router that pools CAS workers and dispatches client connections
3. **CAS + DB server** ([[modules/src|src/]]) — executes SQL, manages storage, transactions, recovery, and HA

## Build modes — same source, three binaries

The same `src/` tree compiles into three binaries via preprocessor guards. See [[Build Modes (SERVER SA CS)]].

## Query path is split across the wire

> [!key-insight] Parser + optimizer run client-side
> The parser and optimizer execute on the **client** (`#if !defined(SERVER_MODE)`). The server only sees the serialized [[components/xasl|XASL]] plan. See [[Query Processing Pipeline]] for the full path.

## Key subsystems (in [[modules/src|src/]])

- Query layer: [[components/parser|parser]] + [[components/optimizer|optimizer]] + [[components/query|query (XASL execution)]] + [[components/xasl|xasl]]
- Storage: [[components/storage|page buffer + heap + B-tree + LOB ES]]
- Transactions: [[components/transaction|MVCC + WAL + lock manager + recovery]]
- Schema / catalog / auth: [[components/object|object/]]
- Client API + DB_VALUE: [[components/compat|compat/]]
- Stored procedures: [[components/sp|src/sp]] + [[modules/pl_engine|pl_engine]]
- Connection broker (CAS): [[components/broker-impl|src/broker]] (vs the [[modules/broker|top-level broker/]] CMake target)
- Threads / daemons: [[components/thread|thread/]]
- Error / memory / lock-free / porting: [[components/base|base/]]

## Navigation

- [[modules/_index|Modules]] — per-directory summaries
- [[components/_index|Components]] — subsystems (optimizer, lock mgr, page buffer, …)
- [[flows/_index|Flows]] — request paths and lifecycles
- [[decisions/_index|Decisions]] — ADRs
- [[dependencies/_index|Dependencies]] — external libs
- [[Tech Stack]] · [[Data Flow]] · [[Dependency Graph]] · [[Key Decisions]]
