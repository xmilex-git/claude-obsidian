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

## 3rdparty library graph

All libraries downloaded from `github.com/CUBRID/3rdparty` mirror at configure time. Static on Linux; prebuilt DLLs on Windows.

```
libcubrid.so ──► libexpat.a       (XML)
             ──► libjansson.a     (JSON DOM)
             ──► libssl.a + libcrypto.a  (TLS — OpenSSL 1.1.1w EOL)
             ──► libodbc.so       (ODBC — unixODBC 2.3.9, shared)
             ──► liblz4.a         (compression)
             ──► libre2.a         (regex)
             ──► [headers] rapidjson  (header-only JSON)
             ──► libtbb.a         (parallel query — SERVER_MODE only)

csql ────────► libedit.a          (line editing — Linux only)

Build tools (not linked):
             ──► flex + bison     (grammar → C++ codegen)
```

See [[dependencies/_index]] for per-library pages. See [[modules/3rdparty]] for CMake build integration.
