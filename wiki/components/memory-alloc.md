---
type: component
parent_module: "[[components/base|base]]"
path: "src/base/"
status: developing
purpose: "Per-thread and per-arena memory allocation, RAII C++ block types, and the SERVER_MODE memory monitoring wrapper — the allocation layer every engine module uses"
key_files:
  - "memory_alloc.c/h (db_private_alloc/free/realloc, free_and_init, os_malloc, alignment macros)"
  - "memory_cwrapper.h (C-compatible; safe to include in headers)"
  - "memory_wrapper.hpp (MUST be last #include in .cpp files; SERVER_MODE global new/delete override)"
  - "area_alloc.c/h (slab allocator for fixed-size objects; lock-free bitmap per block)"
  - "mem_block.hpp/cpp (cubmem::extensible_block, stack_block, pluggable block_allocator)"
  - "memory_private_allocator.hpp/cpp (STL-compatible cubmem::private_allocator<T>)"
  - "fixed_alloc.c / fixed_size_allocator.hpp (heaplayers fixed-size + C++17 template freelist)"
  - "memory_hash.c/h (MHT_TABLE chaining hash with LRU)"
  - "memory_monitor_*.cpp/hpp (per-file:line stats, 16-byte MMON_METAINFO per block)"
tags:
  - component
  - cubrid
  - memory
related:
  - "[[components/base|base]]"
  - "[[Memory Management Conventions]]"
  - "[[components/error-manager|error-manager]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `memory_alloc` — Memory Allocation Subsystem

Implementation of [[Memory Management Conventions]] for the CUBRID engine. Provides per-thread private heaps, slab allocators, C++ RAII block types, STL-compatible allocators, and SERVER_MODE memory monitoring via global operator overrides.

## Allocation Hierarchy

```
memory_wrapper.hpp         ← SERVER_MODE: overrides global new/delete (LAST INCLUDE ONLY)
        │
memory_cwrapper.h          ← SERVER_MODE: overrides malloc/free/calloc/realloc (header-safe)
        │
db_private_alloc           ← per-thread LEA heap (SERVER), db_ws_alloc workspace (CS), conditional (SA)
area_alloc                 ← slab allocator for parse nodes, set elements, fixed-size objects
db_ostk_alloc              ← obstack (stack-like bump allocator, bulk free only)
```

## Core Allocation Functions

### `db_private_alloc` / `db_private_free` / `db_private_realloc`

The primary allocation path for engine server code:

```c
// In debug builds — tracks file/line for leak detection
db_private_alloc(thread_p, size)      // expands to db_private_alloc_debug(...)
db_private_free(thread_p, ptr)
db_private_realloc(thread_p, ptr, size)
```

In SERVER_MODE, these use the per-thread LEA (Low-fragmentation Extended Allocator) heap from `heaplayers/`. In CS_MODE, they delegate to `db_ws_alloc` (workspace allocator). In SA_MODE, conditional on context.

> [!key-insight] Always pass `thread_p`
> `db_private_alloc` requires a `THREAD_ENTRY *` because allocation is per-thread. Never pass NULL except in very early boot code. Passing NULL falls back to global malloc, bypassing tracking.

### `free_and_init(ptr)` — The Canonical Free

```c
#define free_and_init(ptr) \
  do { \
    free((void*)(ptr)); \
    (ptr) = NULL; \
  } while (0)

#define db_private_free_and_init(thrd, ptr) \
  do { \
    db_private_free((thrd), (ptr)); \
    (ptr) = NULL; \
  } while (0)
```

> [!warning] Never use bare `free()`
> Project anti-pattern: bare `free()` is banned. Always use `free_and_init()` or `db_private_free_and_init()`. The nullification step prevents use-after-free bugs from manifesting as mysterious crashes instead of immediate NULL-dereference assertions.

### OS-level Allocation (`os_malloc` / `os_free`)

For allocations that must bypass the private heap (e.g., OS-layer structures):

```c
// CS_MODE / SA_MODE: thin wrappers around malloc/free
os_malloc(size)
os_free(ptr)
os_realloc(ptr, size)

// SERVER_MODE (debug): tracked with resource_tracker
// SERVER_MODE (release): tracked without file/line
os_malloc_debug(size, rc_track, __FILE__, __LINE__)
os_free_debug(ptr, rc_track, __FILE__, __LINE__)
```

## `memory_wrapper.hpp` — The Placement Rule

> [!warning] CI-enforced placement rule
> `memory_wrapper.hpp` **MUST** be the absolute last `#include` in any `.cpp` file. It must never appear in a header file. The required comment above it is:
> ```cpp
> // XXX: SHOULD BE THE LAST INCLUDE HEADER
> #include "memory_wrapper.hpp"
> ```
> CI fails on any violation. The reason: this header overrides global `operator new` / `operator delete` for memory monitoring. Placement earlier than glibc headers causes conflict with glibc's internal placement-new usage.

### `memory_cwrapper.h` — The Header-Safe Alternative

Overrides C-level `malloc`/`free`/`calloc`/`realloc` as tracked inline functions. Safe to include in headers. Included by `memory_alloc.h` itself, so any file including `memory_alloc.h` gets automatic C-level tracking.

## Slab Allocator (`area_alloc`)

`area_alloc` is a slab-style allocator for frequently-allocated fixed-size objects:
- Used for: parse tree nodes, set elements, and other pool-eligible types
- Internal structure: blocks of slots; each block uses a lock-free bitmap for free-slot tracking
- Thread-safe without locks via CAS operations on the bitmap

```c
AREA *area_create(const char *name, size_t element_size, size_t alloc_cnt);
void *area_alloc(AREA *area);
void  area_free(AREA *area, void *ptr);
void  area_destroy(AREA *area);
```

## C++ Block Types (`mem_block.hpp`, `cubmem` namespace)

| Type | Description |
|------|-------------|
| `cubmem::extensible_block` | Heap-backed block that grows on demand; owns its memory |
| `cubmem::stack_block<N>` | Stack-allocated fixed buffer; falls back to heap if N exceeded |
| `cubmem::block_allocator` | Strategy object: swap between malloc/private heap/obstack backends |
| `cubmem::private_allocator<T>` | STL-compatible allocator backed by per-thread private heap |

```cpp
cubmem::extensible_block buf;
buf.extend_to(1024);   // ensures capacity >= 1024 bytes, reallocs if needed
// buf.ptr / buf.dim are the raw pointer and capacity
```

## Memory Monitoring (`memory_monitor_*.cpp/hpp`)

In SERVER_MODE, each allocation carries a 16-byte `MMON_METAINFO` header inserted before the user pointer:
- Records `__FILE__` + `__LINE__` of the allocation call-site
- Aggregated per file:line into a stats table
- Exposed via server diagnostics

The monitoring overhead is always present in SERVER_MODE (both debug and release), controlled by the metainfo header size.

## Alignment Macros

```c
DB_ALIGN(offset, align)           // round up to next multiple of align (power-of-2)
DB_ALIGN_BELOW(offset, align)     // round down
DB_WASTED_ALIGN(offset, align)    // bytes wasted to reach alignment
PTR_ALIGN(addr, boundary)         // align raw pointer (debug: zeroes waste bytes)
MAX_ALIGNMENT                     // DOUBLE_ALIGNMENT (8 bytes)
PTR_ALIGNMENT                     // 4 bytes (32-bit) or 8 bytes (64-bit)
```

## Private Heap Lifecycle

```c
HL_HEAPID db_create_private_heap(void);
void      db_clear_private_heap(THREAD_ENTRY *thread_p, HL_HEAPID heap_id);
void      db_destroy_private_heap(THREAD_ENTRY *thread_p, HL_HEAPID heap_id);
HL_HEAPID db_replace_private_heap(THREAD_ENTRY *thread_p);  // swap in new heap
HL_HEAPID db_change_private_heap(THREAD_ENTRY *thread_p, HL_HEAPID heap_id);
```

Each server thread owns one private heap created at thread initialization. The heap is cleared (not freed) between requests to amortize allocation overhead.

## Related

- [[Memory Management Conventions]] — project-wide convention this implements
- [[components/base|base]] — parent hub
- [[components/page-buffer|page-buffer]] — heavy consumer of db_private_alloc
- [[Build Modes (SERVER SA CS)]] — controls which wrapper variants activate
- Source: [[sources/cubrid-src-base|cubrid-src-base]]
