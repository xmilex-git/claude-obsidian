---
type: concept
title: "Memory Management Conventions"
status: developing
tags:
  - concept
  - cubrid
  - memory
  - convention
related:
  - "[[CUBRID]]"
  - "[[Code Style Conventions]]"
  - "[[Error Handling Convention]]"
  - "[[components/base|base]]"
  - "[[components/heaplayers|heaplayers]]"
created: 2026-04-23
updated: 2026-04-23
---

# Memory Management Conventions

CUBRID enforces a **C error-and-ownership model** even though `.c` files compile as C++17. RAII and exceptions are forbidden in engine code.

## Allocation APIs

| API | Lifetime / scope | Use when |
|-----|------------------|----------|
| `malloc` / `free_and_init` | process | general C allocations (always pair with `free_and_init` to nullify the pointer) |
| `db_private_alloc(thread_p, size)` | per-thread / server-side | server-internal allocations tied to a worker thread |
| `parser_alloc(parser, len)` | parser | strings/nodes that live as long as the parser context |

## Hard rules

> [!warning] Never call `free()` directly
> Always use `free_and_init(ptr)` so the pointer becomes `NULL` after release. Bare `free` is an anti-pattern.

> [!warning] No C++ exceptions, no RAII
> Engine code uses the C error model: `er_set(...)` + return codes (see [[Error Handling Convention]]). RAII smart pointers and `try/catch` are not used in `.c`/`.cpp` engine files.

## `memory_wrapper.hpp` rule

`memory_wrapper.hpp` MUST be the **last** include in any file that uses it, with the comment:

```cpp
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"
```

Skipping the comment is a CI-flagged anti-pattern.

## Embedded heap allocators

`src/heaplayers/` contains an embedded malloc/heap allocator suite (3rd-party). Notable: `lea_heap.c` is a 181 KB malloc implementation — **do not modify**.

## Related

- Source: [[cubrid-AGENTS]]
- Components: [[components/base|src/base]], [[components/heaplayers|src/heaplayers]]
