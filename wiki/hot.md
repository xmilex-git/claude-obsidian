---
type: meta
title: "Hot Cache"
updated: 2026-04-23T20:00:00
tags:
  - meta
  - hot
status: active
---

# Recent Context

## Last Updated
2026-04-23. First CUBRID ingest complete: `AGENTS.md` (project guide). 14 new pages + 5 hub updates.

## Key Recent Facts
- CUBRID = open-source C/C++17 RDBMS, Apache 2.0, v11.5.x. Java PL engine for stored procedures.
- `.c` files compiled as **C++17** (`c_to_cpp.sh`). Intentional, not migration.
- Same source → 3 binaries via preprocessor guards: [[Build Modes (SERVER SA CS)]] — `cub_server`, `cubridsa`, `cubridcs`.
- **Parser + optimizer run client-side** (`#if !defined(SERVER_MODE)`). Server only sees serialized [[components/xasl|XASL]].
- Two `broker/` directories: top-level [[modules/broker|`broker/`]] = CMake target; [[components/broker-impl|`src/broker/`]] = implementation.
- Engine code uses **C error model** ([[Error Handling Convention]]) and **no RAII** ([[Memory Management Conventions]]).
- Adding an error code touches **6 files**.
- `csql_grammar.y` is 646 KB.

## Recent Changes
- Created entity: [[CUBRID]]
- Created concepts: [[Query Processing Pipeline]], [[Build Modes (SERVER SA CS)]], [[Memory Management Conventions]], [[Error Handling Convention]], [[Code Style Conventions]]
- Created modules: [[modules/src|src]], [[modules/broker|broker]], [[modules/pl_engine|pl_engine]], [[modules/unit_tests|unit_tests]]
- Created components: [[components/parser]], [[components/optimizer]], [[components/storage]], [[components/transaction]]
- Created source summary: [[cubrid-AGENTS]]
- Updated hubs: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Key Decisions]], [[index]]
- Symlinked source tree: `.raw/cubrid -> /Users/song/DEV/cubrid` (so future ingests use `.raw/cubrid/...` paths without duplication)

## Active Threads
- Next ingest candidates (queued): `src/AGENTS.md`, `pl_engine/AGENTS.md`, `unit_tests/AGENTS.md`, then `CMakeLists.txt`, `build.sh`, then per-module deeper passes (`broker/`, `cs/`, `cm_common/`).
- User stopped further ingest after AGENTS.md → moving to git remote + Obsidian Git plugin setup.
