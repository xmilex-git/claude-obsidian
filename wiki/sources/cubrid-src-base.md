---
status: active
created: 2026-04-23
type: source
title: "CUBRID src/base/ — Core Utilities & Infrastructure"
source_path: "src/base/"
date_ingested: 2026-04-23
tags:
  - source
  - cubrid
  - base
  - error-handling
  - memory
  - lock-free
  - porting
related:
  - "[[components/base|base]]"
  - "[[components/error-manager|error-manager]]"
  - "[[components/memory-alloc|memory-alloc]]"
  - "[[components/lockfree|lockfree]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[components/porting|porting]]"
  - "[[Error Handling Convention]]"
  - "[[Memory Management Conventions]]"
---

# Source: `src/base/` — Core Utilities & Infrastructure

**Date ingested:** 2026-04-23
**Primary files read:** `AGENTS.md`, `error_manager.h`, `memory_alloc.h`, `porting.h`, `lockfree_hashmap.hpp`, `lockfree_transaction_system.hpp`, `system_parameter.h`

## Summary

`src/base/` is the foundational utility layer of CUBRID. Every other engine module depends on it. It provides 13 distinct subsystems grouped under four primary topics assigned to this directory:

1. **Error Handling** — `error_code.h` (~1700 codes), `error_manager.c/h`, `error_context.hpp`
2. **Memory Management** — `memory_alloc.c/h`, `area_alloc`, C++ `cubmem` namespace, `memory_wrapper.hpp`, `memory_monitor_*`
3. **Lock-Free Data Structures** — modern `lockfree::hashmap` + legacy `LF_HASH_TABLE` family, epoch-based reclamation
4. **Platform Abstraction / Porting** — `porting.h` (POSIX↔Win32), `dynamic_load`, `process_util`

Additional subsystems: i18n/locale, timezone, performance monitoring, serialization/packing, data structures (heap, rb-tree, dynamic array), system configuration (`system_parameter`), cryptography, logging/diagnostics, and miscellaneous utilities.

## Key Findings

- `memory_wrapper.hpp` carries a **CI-enforced placement rule**: it must be the absolute last `#include` in any `.cpp` file and cannot appear in headers at all. CI fails on violations.
- Error codes are always negative integers; `NO_ERROR = 0`. Adding a new code requires six-place update (see root AGENTS.md).
- The lock-free layer has two generations: legacy C (`lock_free.c`, `LF_HASH_TABLE`) and modern C++ (`lockfree::hashmap<Key,T>` with `lockfree::tran::system`). The modern layer solves ABA via epoch-based retirement; nodes are only reclaimed once all concurrent readers have advanced their transaction IDs.
- `system_parameter.h` defines ~400 `PRM_ID_*` enum values (the largest file in base). Referenced throughout all modules via `prm_get_*_value(PRM_ID_...)`.
- `porting.h` provides `#define` shims mapping POSIX calls to Win32 equivalents (e.g., `sleep`, `snprintf`, `stat`, `lseek`) when `WINDOWS` is defined.

## Pages Created

- [[components/base|base]] — component hub
- [[components/error-manager|error-manager]] — er_set, severity, macros
- [[components/memory-alloc|memory-alloc]] — db_private_alloc, free_and_init, wrapper rule
- [[components/lockfree|lockfree]] — lock-free hashmap + transaction system
- [[components/system-parameter|system-parameter]] — PRM_ID_* registry
- [[components/porting|porting]] — OS portability layer
