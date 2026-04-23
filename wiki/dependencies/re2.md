---
type: dependency
name: "RE2"
version: "2022-06-01 snapshot"
source: "https://github.com/CUBRID/3rdparty/raw/develop/re2/2022-06-01.tar.gz"
license: "BSD-3-Clause"
bundled: true
used_by:
  - "Regular expression evaluation in SQL (REGEXP / RLIKE operators)"
risk: low
tags:
  - dependency
  - cubrid
  - regex
  - sql
created: 2026-04-23
updated: 2026-04-23
---

# RE2

## What it does

RE2 is Google's regular expression library that uses finite automata (NFA/DFA) rather than backtracking. It guarantees linear-time matching and avoids catastrophic backtracking vulnerabilities (ReDoS), at the cost of not supporting backreferences or lookahead/lookbehind.

## Why CUBRID uses it

CUBRID uses RE2 for the `REGEXP` / `RLIKE` SQL operators. RE2's linear-time guarantee is important for a database engine where user-supplied regex patterns could otherwise cause denial-of-service via catastrophic backtracking.

## Integration points

- CMake target: `re2`
- Linux: built from source via `ExternalProject_Add` using the RE2 Makefile (`BUILD_IN_SOURCE true`)
- No `configure` step; build flags: `CFLAGS=-fPIC CXXFLAGS=-fPIC`
- Produces `obj/libre2.a` in the source directory
- Windows: prebuilt `re2.dll`/`re2.lib` from `win/3rdparty/RE2/`; DLL copied to runtime output directory
- Only `EXTERNAL` mode is supported; `SYSTEM` raises a `FATAL_ERROR`
- Exposes `RE2_LIBS` and `RE2_INCLUDES` via `expose_3rdparty_variable(RE2)`
- Included in `EP_TARGETS`, `EP_LIBS`, `EP_INCLUDES` (all targets)

## Risk / notes

- Version is a date-based snapshot (`2022-06-01`), not a tagged release. RE2 uses date-based versioning.
- BSD-3-Clause — permissive, no concerns.
- Only `EXTERNAL` mode is supported — cannot be overridden to system RE2.
- `BUILD_IN_SOURCE true` — clean rebuilds require clearing ExternalProject source.

## Related

- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
