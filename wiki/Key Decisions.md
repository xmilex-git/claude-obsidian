---
type: overview
title: "CUBRID Key Decisions"
updated: 2026-04-23
tags:
  - cubrid
  - decision
  - overview
status: stub
related:
  - "[[Architecture Overview]]"
  - "[[decisions/_index|Decisions]]"
---

# CUBRID Key Decisions

> Stub. ADR entries filed under [[decisions/_index|decisions/]] will be summarized here.

Candidates to file during ingest:

- Three-process-group topology (client / broker / server)
- MVCC adoption and its transaction isolation semantics
- HA design: who replays what, failover model
- Page / heap / B+tree storage choices
- PL engine introduction ([[pl_engine]])
- Build system migration (autotools → CMake)
