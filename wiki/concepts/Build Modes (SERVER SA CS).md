---
type: concept
title: "Build Modes (SERVER / SA / CS)"
status: developing
tags:
  - concept
  - cubrid
  - build
related:
  - "[[CUBRID]]"
  - "[[Tech Stack]]"
  - "[[Architecture Overview]]"
  - "[[modules/cubrid|cubrid module]]"
  - "[[modules/sa|sa module]]"
  - "[[modules/cs|cs module]]"
created: 2026-04-23
updated: 2026-04-23
---

# Build Modes — `SERVER_MODE` / `SA_MODE` / `CS_MODE`

CUBRID compiles **the same source tree** into three different binaries by toggling preprocessor guards.

| Guard | Output | CMake target dir | Purpose |
|-------|--------|------------------|---------|
| `SERVER_MODE` | `cub_server` (binary) | [[modules/cubrid|`cubrid/`]] | DB server process |
| `SA_MODE` | `cubridsa` (library) | [[modules/sa|`sa/`]] | Standalone — client + server in one process |
| `CS_MODE` | `cubridcs` (library) | [[modules/cs|`cs/`]] | Client library — connects to remote server |

## Implications

- **Parser + optimizer code is client-side**: wrapped in `#if !defined(SERVER_MODE)`. Not present in `cub_server`.
- **Standalone (`SA_MODE`) is special**: the same process plays both roles. Useful for utilities, recovery tools, single-user mode.
- A change in shared code must be considered against **all three** preprocessor variants.

## Build invocation

```bash
./build.sh -m debug              # debug build (most common)
./build.sh -m release            # release build
./build.sh -m debug -c "-DUNIT_TESTS=ON"  # with unit tests
cmake --preset debug && cmake --build build_preset_debug
```

## Where it shows up

- CMake target dirs: top-level `cubrid/`, `sa/`, `cs/`
- Conditional compilation throughout `src/` (especially `src/parser/`, `src/optimizer/`, `src/query/`)
- The query path: client serializes [[components/xasl|XASL]] → server deserializes & executes ([[Query Processing Pipeline]])

## Related

- Source: [[cubrid-AGENTS]]
