---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/extendible_hash.c"
status: active
purpose: "Disk-resident extendible hash table used internally (e.g. unique constraint checking, join operations); supports search, insert, delete, and map (iterate)"
key_files:
  - "extendible_hash.c — implementation"
  - "extendible_hash.h — public API"
public_api:
  - "ehash_search(thread_p, ehid, key, value_ptr) -> EH_SEARCH"
  - "ehash_insert(thread_p, ehid, key, value_ptr) -> void*"
  - "ehash_delete(thread_p, ehid, key) -> void*"
  - "ehash_map(thread_p, ehid, fun, args)"
tags:
  - component
  - cubrid
  - storage
  - hash
  - internal
related:
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/file-manager|file-manager]]"
created: 2026-04-23
updated: 2026-04-23
---

# `extendible_hash.c` — Disk-Based Extendible Hash

The extendible hash is a disk-resident dynamic hash structure. It uses two file types: `FILE_EXTENDIBLE_HASH` for bucket pages and `FILE_EXTENDIBLE_HASH_DIRECTORY` for the directory page(s). The directory doubles in size when a bucket overflows and must be split.

## Identifier

`EHID` addresses an extendible hash instance. All operations take `EHID*` as input.

## Operations

| Function | Semantics |
|----------|-----------|
| `ehash_search` | Look up a key; returns `EH_SEARCH` enum (`EH_KEY_FOUND`, `EH_KEY_NOTFOUND`, `EH_ERROR_OCCURRED`) |
| `ehash_insert` | Insert `(key → OID)` |
| `ehash_delete` | Remove a key; returns old value pointer |
| `ehash_map` | Iterate all entries calling `fun(thread_p, key, value, args)` |

## Recovery Functions

All structural changes are logged:

| Function | Purpose |
|----------|---------|
| `ehash_rv_init_bucket_redo` | Initialize a new bucket page |
| `ehash_rv_init_dir_redo` | Initialize a new directory page |
| `ehash_rv_insert_redo / _undo` | Redo/undo an insert |
| `ehash_rv_delete_redo / _undo` | Redo/undo a delete |
| `ehash_rv_increment` | Redo directory depth increment |
| `ehash_rv_connect_bucket_redo` | Redo: connect a new bucket to the directory |
| `ehash_rv_init_dir_new_page_redo` | Redo: initialize directory expansion page |

## Internal Use

Extendible hash is used internally by the query engine and uniqueness checking subsystems. It is not exposed directly to SQL users. The `FILE_EXTENDIBLE_HASH` and `FILE_EXTENDIBLE_HASH_DIRECTORY` file types in `file_manager.h` are reserved for it.

## Related

- Parent: [[components/storage|storage]]
- [[components/page-buffer]] — directory and bucket pages accessed via buffer pool
- [[components/file-manager]] — allocates pages from `FILE_EXTENDIBLE_HASH*` file types
