---
type: overview
title: "CUBRID Architecture Overview"
updated: 2026-04-23
tags:
  - cubrid
  - architecture
  - overview
status: stub
related:
  - "[[modules/_index|Modules]]"
  - "[[Tech Stack]]"
  - "[[Data Flow]]"
  - "[[Dependency Graph]]"
  - "[[Key Decisions]]"
---

# CUBRID Architecture Overview

> Stub. Will be expanded as `ingest` runs populate [[modules/_index|modules]], [[components/_index|components]], [[flows/_index|flows]], and [[decisions/_index|decisions]].

## At a glance

CUBRID is an open-source, ACID-compliant relational database management system written primarily in C/C++. Source tree: `/Users/song/DEV/cubrid/`.

Three-process-group topology:

1. **Client / driver** (cubrid-cci, cubrid-jdbc, client libs in `cs/`)
2. **Broker** (`broker/`) — multi-threaded connection router that pools clients and dispatches to CAS workers
3. **CAS + DB server** (`src/`) — executes SQL, manages storage, transactions, recovery, and HA

## Key subsystems

See [[modules/_index|Modules]] for the full list. Major areas to expect in `src/`:
- Parser + optimizer + executor (query layer)
- Storage manager (page buffer, heap, B+tree)
- Transaction + log manager
- Lock manager + MVCC
- HA / replication
- PL engine ([[pl_engine]])

## Navigation

- [[modules/_index|Modules]] — per-directory summaries
- [[components/_index|Components]] — subsystems (optimizer, lock mgr, page buffer, …)
- [[flows/_index|Flows]] — request paths and lifecycles
- [[decisions/_index|Decisions]] — ADRs
- [[dependencies/_index|Dependencies]] — external libs
- [[Tech Stack]] · [[Data Flow]] · [[Dependency Graph]] · [[Key Decisions]]
