---
type: component
parent_module: "[[modules/src|src]]"
path: "src/base/"
status: developing
purpose: "Foundational utility layer used by every CUBRID engine module: error handling, memory, lock-free data structures, platform abstraction, i18n, serialization, performance monitoring, and more"
key_files:
  - "error_code.h (~1700 error codes as negative #define constants)"
  - "error_manager.c/h (er_set, error stack, severity, ASSERT_ERROR macros)"
  - "memory_alloc.c/h (db_private_alloc, free_and_init, alignment macros)"
  - "memory_wrapper.hpp (MUST be last #include — SERVER_MODE global new/delete override)"
  - "memory_cwrapper.h (C-safe header-includable wrapper)"
  - "lockfree_hashmap.hpp/cpp (modern C++ lock-free hashmap)"
  - "lock_free.c/h (legacy C lock-free hash + freelist)"
  - "system_parameter.c/h (~400 PRM_ID_* params, reads cubrid.conf)"
  - "porting.c/h (POSIX/Win32 function mapping, platform constants)"
  - "area_alloc.c/h (slab allocator for fixed-size objects)"
  - "mem_block.hpp/cpp (cubmem RAII blocks)"
tags:
  - component
  - cubrid
  - base
  - error-handling
  - memory
  - lock-free
  - porting
related:
  - "[[modules/src|src]]"
  - "[[components/error-manager|error-manager]]"
  - "[[components/memory-alloc|memory-alloc]]"
  - "[[components/lockfree|lockfree]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[components/porting|porting]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[Error Handling Convention]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/base/` — Core Utilities & Infrastructure

This directory is the absolute foundation of CUBRID. Every other engine module (`storage`, `query`, `transaction`, `parser`, `thread`, etc.) includes headers from here. It provides 13 subsystems, four of which are its primary responsibility.

## Architecture Overview

```
src/base/ (foundational layer — no engine deps upward)
         │
         ├── Error Handling          error_code.h + error_manager + error_context
         │
         ├── Memory Management       memory_alloc + area_alloc + cubmem namespace
         │       │
         │       └── memory_wrapper.hpp  ← LAST include, SERVER_MODE only
         │
         ├── Lock-Free Structures    lockfree::hashmap (modern) + LF_HASH_TABLE (legacy)
         │       │
         │       └── lockfree::tran  ← epoch-based safe reclamation
         │
         ├── Platform Abstraction    porting.h (POSIX↔Win32 shims)
         │
         ├── System Configuration    system_parameter.h (~400 PRM_ID_* values)
         │
         └── + 8 more subsystems     i18n, timezone, perf, serialization, crypto, data structures, logging, misc
```

## Primary Subsystems

### 1. Error Handling

Implementation: [[components/error-manager|error-manager]]
Concept: [[Error Handling Convention]]

`error_code.h` defines ~1700 error codes as `#define` negative integers. `error_manager.c/h` implements the thread-local error stack, severity routing, and logging. All code calls `er_set(severity, ARG_FILE_LINE, ER_CODE, nargs, ...)`.

### 2. Memory Management

Implementation: [[components/memory-alloc|memory-alloc]]
Concept: [[Memory Management Conventions]]

Three allocation tiers:
- `db_private_alloc` — per-thread LEA heap (SERVER_MODE) or workspace (CS_MODE)
- `area_alloc` — slab allocator for parse nodes, set elements
- `db_ostk_alloc` — obstack (stack-like bump allocator)

`memory_wrapper.hpp` overrides global `new`/`delete` for memory monitoring in SERVER_MODE; **must be the last `#include`** in any `.cpp` file (CI-enforced).

### 3. Lock-Free Data Structures

Implementation: [[components/lockfree|lockfree]]

Two generations:
- **Modern**: `lockfree::hashmap<Key,T>`, `lockfree::freelist`, `lockfree::tran::system` (C++17)
- **Legacy**: `LF_HASH_TABLE`, `LF_FREELIST`, `LF_TRAN_SYSTEM` (C API, same concepts)

Used by [[components/parallel-task-queue|parallel-task-queue]] and internal caches.

### 4. Platform Abstraction

Implementation: [[components/porting|porting]]

`porting.h` provides `#define` shims mapping POSIX calls to Win32 equivalents. Also provides portable atomic ops, `timeval` helpers, and numeric constants (`ONE_K`, `ONE_M`, etc.).

### 5. System Configuration

Implementation: [[components/system-parameter|system-parameter]]

~400 `PRM_ID_*` enum values. Every component reads its tuning knobs via `prm_get_bool_value(PRM_ID_...)`, `prm_get_integer_value(...)`, etc.

## Additional Subsystems (brief)

| Subsystem | Key Files | Notes |
|-----------|-----------|-------|
| Internationalization | `intl_support.c`, `language_support.c`, `uca_support.c` | UTF-8/EUC-KR, UCA collation |
| Timezone | `tz_support.c`, `tz_compile.c` | IANA DB compiler + runtime conversion |
| Performance Monitoring | `perf_monitor.c`, `perf.hpp` (cubperf namespace) | Legacy C + modern C++ |
| Serialization | `packer.hpp` (cubpacking namespace), `packable_object.hpp` | Client→server XASL transport |
| Data Structures | `binaryheap.c`, `rb_tree.h`, `resource_shared_pool.hpp` | Binary heap for top-N, red-black tree |
| Cryptography | `encryption.c`, `sha1.c`, `base64.c`, `CRC.h` | Auth + integrity |
| Logging/Diagnostics | `ddl_log.c`, `fault_injection.c`, `resource_tracker.hpp` | DDL audit, debug fault injection |
| Misc Utilities | `scope_exit.hpp`, `base_flag.hpp`, `compressor.hpp` | C++17 RAII + LZ4 |

## Memory Allocation Hierarchy

```
memory_wrapper.hpp         (SERVER_MODE: overrides global new/delete — LAST INCLUDE)
        │
memory_cwrapper.h          (SERVER_MODE: overrides malloc/free — safe in headers)
        │
db_private_alloc           (per-thread LEA heap, SERVER / SA / CS conditional)
area_alloc                 (slab, fixed-size objects)
db_ostk_alloc              (obstack, stack-like)
```

## Lock-Free Layer Hierarchy

```
lockfree::hashmap<Key,T>   (modern C++ API)
        │
lockfree::freelist<T>      (node recycling pool)
        │
lockfree::tran::table      (per-structure transaction table)
        │
lockfree::tran::descriptor (per-thread: active tx id + retired node list)
        │
lockfree::tran::system     (index manager: assigns descriptor slots)
        │
lockfree::bitmap           (CAS-based index allocation)
```

Legacy equivalent: `LF_HASH_TABLE` → `LF_FREELIST` → `LF_TRAN_SYSTEM` → `LF_TRAN_ENTRY`

## Quick Reference

| Task | Primary File(s) |
|------|-----------------|
| Add error code | `error_code.h` — 6-place update (see root AGENTS.md) |
| Fix memory leak | `memory_alloc.c`, `resource_tracker.hpp` |
| Fix lock-free structure | `lockfree_hashmap.hpp` (modern), `lock_free.c` (legacy) |
| Add system parameter | `system_parameter.c/h` — add `PRM_ID_*`, update `prm_Def`, `PRM_LAST_ID` |
| Fix charset/collation | `intl_support.c`, `language_support.c`, `uca_support.c` |
| Add perf counter | `perf_monitor.c` (legacy) or `perf.hpp` (modern) |

## Related

- Parent: [[modules/src|src]]
- [[components/page-buffer|page-buffer]] — uses memory + error subsystems heavily
- [[components/parallel-task-queue|parallel-task-queue]] — uses lock-free CAS structures
- [[Build Modes (SERVER SA CS)]] — controls which memory wrappers activate
- Source: [[sources/cubrid-src-base|cubrid-src-base]]
