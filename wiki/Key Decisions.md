---
type: overview
title: "CUBRID Key Decisions"
updated: 2026-04-23
tags:
  - cubrid
  - decision
  - overview
status: developing
related:
  - "[[Architecture Overview]]"
  - "[[decisions/_index|Decisions]]"
---

# CUBRID Key Decisions

> Stub. ADR entries filed under [[decisions/_index|decisions/]] will be summarized here.

## Decisions surfaced from [[cubrid-AGENTS|AGENTS.md]] (2026-04-23)

- **`.c` files compiled as C++17** (`c_to_cpp.sh`) — explicit project policy, not a migration in progress.
- **One source tree → three binaries via preprocessor guards** — see [[Build Modes (SERVER SA CS)]]. Same source produces `cub_server`, `cubridsa`, `cubridcs`.
- **Parser & optimizer are client-side** — XASL exists as a serializable IR specifically because the server must receive a finalized plan, not source text.
- **C error model in C++ code** — no exceptions, no RAII for memory in engine code. See [[Error Handling Convention]] and [[Memory Management Conventions]].
- **`memory_wrapper.hpp` MUST be the last include** — enforced by CI, with required marker comment.
- **Header guards `_FILENAME_H_`, NOT `#pragma once`** — CI-checked.
- **Large files (10K+ lines) are intentional and must NOT be split** — `csql_grammar.y` (646 KB), various `.c` files. Project policy.

## Candidates to file (need source)

- MVCC adoption and isolation semantics
- HA design: who replays what, failover model
- Page / heap / B-tree storage choices
- Build system migration (autotools → CMake)
- PL engine introduction ([[modules/pl_engine|pl_engine]])

Each will get an ADR under [[decisions/_index|decisions/]] as deeper modules are ingested.
