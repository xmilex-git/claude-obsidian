---
type: overview
title: "CUBRID Dependency Graph"
updated: 2026-04-23
tags:
  - cubrid
  - dependency
  - overview
status: stub
related:
  - "[[Architecture Overview]]"
  - "[[dependencies/_index|Dependencies]]"
  - "[[Tech Stack]]"
---

# CUBRID Dependency Graph

> Partially populated. `3rdparty/` ingested 2026-04-23. Submodules still pending.

## Internal dependency sketch

```
cubrid-cci ──► (C client interface)
cubrid-jdbc ─► (JDBC driver, depends on CCI)

Client libs (cs/) ──► Broker (broker/) ──► CAS ──► DB server (src/)
                                                   ├── pl_engine/
                                                   ├── cm_common/
                                                   └── 3rdparty/*
cubridmanager (Java GUI) ── depends on cm_common + shell tools
```

See [[dependencies/_index]] for per-library pages.
