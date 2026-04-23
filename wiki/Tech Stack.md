---
type: overview
title: "CUBRID Tech Stack"
updated: 2026-04-23
tags:
  - cubrid
  - stack
  - overview
status: developing
related:
  - "[[Architecture Overview]]"
  - "[[dependencies/_index|Dependencies]]"
---

# CUBRID Tech Stack

## Languages

- **C** — most engine source (`.c`)
- **C++17** — `.c` files are also compiled as C++17 (`c_to_cpp.sh`); newer subsystems and tooling are `.cpp`
- **Java** — [[modules/pl_engine|PL engine]], [[modules/cubrid-jdbc|JDBC driver]], [[modules/cubridmanager|CUBRID Manager]]
- **SQL / CSQL** — query language and utility interpreter
- **Bison / Flex** — `csql_grammar.y` (646 KB), `csql_lexer.l`, plus loaddb grammar

## Build

- **CMake** (`CMakeLists.txt`, `cmake/`, `CMakePresets.json`)
- **`build.sh`** wrapper — see [[Build Modes (SERVER SA CS)]] and [[cubrid-AGENTS|AGENTS.md]] for flag table
- **Gradle** for [[modules/pl_engine|pl_engine]]; **Ant** for [[modules/cubrid-jdbc|cubrid-jdbc]]
- Targets: Linux (primary), Windows, macOS (partial), Debian packaging in `debian/`
- Toolchain: GCC 8+ (devtoolset-8 recommended), JDK 1.8+, CMake 3.21+

## Runtime / platform

- POSIX threads, shared memory, mmap
- Multi-process: client / [[modules/broker|broker]] / CAS workers / DB server
- HA (replication) — design TBD

## Conventions worth carrying

- [[Memory Management Conventions]] — `free_and_init`, `db_private_alloc`, `parser_alloc`; no RAII
- [[Error Handling Convention]] — C-style codes, no C++ exceptions
- [[Code Style Conventions]] — 2-space indent, 120 col, GNU braces

## CI

- **GitHub Actions** (`.github/workflows/check.yml`) — license headers, PR title regex, code style, cppcheck, memory_wrapper check
- **Jenkins** (`Jenkinsfile`) — primary build (release/debug parallel), Docker `cubridci/cubridci:develop`
- **CircleCI** (`.circleci/config.yml`) — SQL tests (10× parallel), shell tests (50× parallel)

## Bundled (`3rdparty/`)

Populated by future ingest. See [[dependencies/_index]].

## Submodules (`.gitmodules`)
- [[cubrid-cci]]
- [[cubrid-jdbc]]
- [[cubridmanager]]
