---
type: entity
entity_type: product
name: "CUBRID"
license: "Apache 2.0"
language: "C/C++17, Java"
version: "11.5.x"
status: developing
tags:
  - entity
  - product
  - rdbms
  - cubrid
related:
  - "[[Architecture Overview]]"
  - "[[Tech Stack]]"
  - "[[Query Processing Pipeline]]"
  - "[[modules/src|src module]]"
  - "[[modules/pl_engine|pl_engine]]"
created: 2026-04-23
updated: 2026-04-23
---

# CUBRID

Open-source object-relational database management system (RDBMS).

## Identity

- **Type:** RDBMS with object-relational extensions
- **License:** Apache 2.0
- **Languages:** C/C++17 (engine), Java (PL engine, JDBC, Manager), Python/PHP/Perl (contrib drivers)
- **Version:** 11.5.x (current at time of ingest)
- **Source:** `~/dev/cubrid/`

## Architecture in one line

Three-process model: **client (CCI/JDBC) → broker (CAS pool) → DB server**, with a parser/optimizer that runs **client-side**.

## Distinctive technical traits

- All `.c` files compiled as **C++17** (`c_to_cpp.sh`)
- Same source produces 3 binaries via preprocessor guards: `SERVER_MODE` / `SA_MODE` / `CS_MODE`
- Java PL engine (`pl_engine/`) for stored procedures, JNI-bridged from C++ engine (`src/sp/`)
- MVCC + WAL recovery
- Custom `XASL` (eXecutable Algebraic Statement Language) — query plans serialized client → server
- Bison/flex grammar (`csql_grammar.y` is 646 KB)
- Custom error model: negative error codes, `er_set(...)` + return codes; **no C++ exceptions in engine**

## Submodules

- [[cubrid-jdbc]] — JDBC driver (Ant build)
- [[cubrid-cci]] — C Client Interface (CMake build)
- [[cubridmanager]] — CUBRID Manager server

## Build & CI

- Build: CMake + `build.sh` wrapper. GCC 8+ / JDK 1.8+ / CMake 3.21+ required.
- CI: GitHub Actions (lint), Jenkins (build), CircleCI (SQL/shell tests, 50× parallel).

## See also

- Source guide: [[cubrid-AGENTS|AGENTS.md]]
- Hubs: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Key Decisions]]
- Modules: [[modules/_index|all modules]]
