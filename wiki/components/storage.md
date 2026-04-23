---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/"
status: active
purpose: "Buffer pool, heap files, B-tree, external storage"
key_files:
  - "page_buffer.c (LRU buffer pool, dirty tracking)"
  - "page_buffer.h (PAGE_BUFFER struct)"
  - "btree.c (B-tree: btree_find, btree_range_search)"
  - "es.c (external storage backend for LOBs)"
public_api: []
tags:
  - component
  - cubrid
  - storage
  - server
related:
  - "[[modules/src|src]]"
  - "[[components/transaction|transaction]]"
  - "[[components/object|object]]"
  - "[[Architecture Overview]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/storage/` — Storage Layer

On-disk + in-memory storage primitives: buffer pool, heap files, B-tree indexes, and the LOB external storage backend.

## Pieces

| Piece | File(s) | Notes |
|-------|---------|-------|
| Buffer pool | `page_buffer.c` / `page_buffer.h` | LRU replacement, hash table, dirty tracking — `PAGE_BUFFER` struct |
| B-tree | `btree.c` | Entries: `btree_find()`, `btree_range_search()` |
| Heap files | (heap_*) | Row-oriented page storage |
| External storage | `es.c` | Backend for LOB payloads (cross-cutting with [[components/object|object/lob_locator.cpp]]) |

## Common modifications (from [[cubrid-AGENTS|AGENTS.md]])

- **Fix index scan** → `btree.c`, entries `btree_find()` / `btree_range_search()`
- **Fix buffer pool** → `page_buffer.c` (LRU replacement, dirty tracking)

## LOB cross-cutting

> [!info] LOB handling spans two components
> The LOB locator lives in [[components/object|`src/object/lob_locator.cpp`]] while the external storage backend is here in `es.c`. Treat changes to LOB behavior as a multi-file edit.

## Related

- Parent: [[modules/src|src]]
- [[components/transaction]] (writes go through WAL)
- Source: [[cubrid-AGENTS]]
