---
type: module
path: "unit_tests/"
status: active
language: "C/C++17"
purpose: "Catch2-based unit tests for engine modules"
test_framework: "Catch2 v2.11.3"
last_updated: 2026-04-23
depends_on:
  - "[[modules/src|src module]]"
used_by: []
tags:
  - module
  - cubrid
  - testing
related:
  - "[[modules/src|src module]]"
created: 2026-04-23
updated: 2026-04-23
---

# `unit_tests/` — Catch2 Unit Tests

C/C++ unit tests using **Catch2 v2.11.3**. Subdirectories mirror engine modules.

## Build

```bash
./build.sh -m debug -c "-DUNIT_TESTS=ON"
```

Unit tests are off by default; opt in with the CMake flag.

## Disabled modules

> [!warning] Compilation issues — disabled in CI
> Some test modules are currently disabled because they don't compile cleanly:
> - `LOCKFREE`
> - `LOADDB`
> - `MEMORY_MONITOR`
>
> If you need coverage in these areas, expect work before tests run.

## Convention

- Add a unit test under `unit_tests/<module>/`
- See `unit_tests/AGENTS.md` (separate ingest) for layout details

## Related

- [[modules/src|src/]] — the code under test
- Source: [[cubrid-AGENTS]]
