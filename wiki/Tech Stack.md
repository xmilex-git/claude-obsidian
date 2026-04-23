---
type: overview
title: "CUBRID Tech Stack"
updated: 2026-04-23
tags:
  - cubrid
  - stack
  - overview
status: stub
related:
  - "[[Architecture Overview]]"
  - "[[dependencies/_index|Dependencies]]"
---

# CUBRID Tech Stack

> Stub. Filled during ingest of `CMakeLists.txt`, `3rdparty/`, `cmake/`, and `.gitmodules`.

## Languages
- C (engine core)
- C++ (newer subsystems, PL engine, tools)
- SQL / CSQL (query language, utility interpreter)
- Java (JDBC driver, CUBRID Manager)

## Build
- CMake (`CMakeLists.txt`, `cmake/`, `CMakePresets.json`)
- `build.sh` wrapper
- Targets: Linux, Windows, macOS (partial), Debian packaging in `debian/`

## Runtime / platform
- POSIX threads, shared memory, mmap
- Optional: MLS / SELinux awareness, HA (3-node replication)

## Bundled (`3rdparty/`)
Populated by ingest. See [[dependencies/_index]].

## Submodules (`.gitmodules`)
- [[cubrid-cci]]
- [[cubrid-jdbc]]
- [[cubridmanager]]
