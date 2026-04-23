---
type: component
parent_module: "[[components/base|base]]"
path: "src/heaplayers/"
status: stable
purpose: "Bundled 3rd-party heap allocator library (Heap Layers by Emery Berger) — provides the per-thread LEA heap consumed by db_private_alloc in SERVER_MODE"
key_files:
  - "lea_heap.c (~181 KB — Doug Lea malloc implementation; DO NOT MODIFY)"
  - "heaplayers.h (master include; HL namespace umbrella)"
  - "utility/all.h, heaps/all.h, locks/all.h, threads/all.h, wrappers/all.h (sub-group headers)"
tags:
  - component
  - cubrid
  - memory
  - third-party
related:
  - "[[components/memory-alloc|memory-alloc]]"
  - "[[Memory Management Conventions]]"
  - "[[components/base|base]]"
created: 2026-04-23
updated: 2026-04-23
---

# `heaplayers` — Embedded Heap Allocator Library (3rd-party)

> [!warning] Do not modify — 3rd-party drop-in
> `src/heaplayers/` is an unmodified vendored copy of **Heap Layers** (Emery Berger, UMass Amherst). It is Apache 2.0 licensed upstream. **Do not change files here.** Any needed fix must be tracked as a vendor-patch decision, not an inline edit.

> [!warning] Excluded from cppcheck
> Per project AGENTS.md, `src/heaplayers/` is **excluded from cppcheck**. Never add inline `// cppcheck-suppress` comments inside these files. The entire directory is on the CI exclusion list.

## What It Is

Heap Layers is a composable, policy-based C++ memory allocator framework described in the PLDI 2001 paper *"Composing High-Performance Memory Allocators"* (Berger, Zorn, McKinley). It exposes a namespace `HL` and organises allocators into layered sub-groups:

| Sub-group header | Contents |
|------------------|----------|
| `utility/all.h` | Utility templates (alignment, stats wrappers) |
| `heaps/all.h` | Concrete heap implementations |
| `locks/all.h` | Lock policies for thread safety |
| `threads/all.h` | Thread-local storage helpers |
| `wrappers/all.h` | Combinator wrappers (size-tracking, etc.) |

The master include is `heaplayers.h`, which pulls in all sub-groups.

## The Key File: `lea_heap.c` (~181 KB)

`lea_heap.c` is a self-contained port of **Doug Lea's dlmalloc** (`malloc-2.8.x`), adapted into the Heap Layers framework. At ~181 KB it is one of the largest files in the CUBRID source tree. It implements the low-fragmentation extended allocator (LEA) that backs each server thread's private heap.

This file is intentionally monolithic — do not attempt to split it.

## Integration with the Engine

Engine code **never calls into `src/heaplayers/` directly.** The boundary is:

```
db_private_alloc(thread_p, size)          ← engine call site (src/base/memory_alloc.c/h)
        │
HL_HEAPID per-thread heap                 ← opaque handle stored in THREAD_ENTRY
        │
lea_heap / Heap Layers internals          ← src/heaplayers/ — this directory
```

The lifecycle functions in `src/base/memory_alloc` (`db_create_private_heap`, `db_clear_private_heap`, `db_destroy_private_heap`) are the only entry points that touch `HL_HEAPID`. All other engine code uses `db_private_alloc` / `db_private_free` / `db_private_realloc`.

In CS_MODE and SA_MODE the per-thread LEA heap is not used; `db_private_alloc` delegates elsewhere (workspace allocator / system malloc). Only SERVER_MODE activates the Heap Layers path.

## Project Rules Summary

| Rule | Detail |
|------|--------|
| Modification | Do not modify any file in `src/heaplayers/` |
| cppcheck | Entire directory excluded from CI cppcheck run |
| `lea_heap.c` | 181 KB Doug Lea malloc — intentionally large, do not split |
| Engine access | Only via `db_private_alloc` family in `src/base/memory_alloc` |
| Build modes | LEA heap active only in SERVER_MODE |

## Related

- [[components/memory-alloc|memory-alloc]] — engine-facing allocator layer; `db_private_alloc` is the sole consumer
- [[Memory Management Conventions]] — project-wide allocation rules
- [[components/base|base]] — parent hub
- [[Build Modes (SERVER SA CS)]] — controls when the LEA path activates
- Source: [[sources/cubrid-src-heaplayers|cubrid-src-heaplayers]]
