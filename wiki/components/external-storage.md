---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/es*.c"
status: active
purpose: "External storage API for LOB (BLOB/CLOB) data: routes create/read/write/delete/copy/rename operations to POSIX filesystem, OWFS, or local backends via a URI-addressed file model"
key_files:
  - "es.c / es.h — unified API: es_create_file, es_write_file, es_read_file, es_delete_file, es_copy_file, es_rename_file"
  - "es_common.c / es_common.h — ES_TYPE enum (OWFS/POSIX/LOCAL), URI prefix constants, hash utilities"
  - "es_posix.c / es_posix.h — POSIX filesystem backend: two-level hashed directory, create/read/write/delete"
  - "es_owfs.c / es_owfs.h — OWFS (distributed filesystem) backend"
public_api:
  - "es_init(uri) — initialize active backend from URI prefix"
  - "es_final()"
  - "es_create_file(out_uri)"
  - "es_write_file(uri, buf, count, offset)"
  - "es_read_file(uri, buf, count, offset)"
  - "es_delete_file(uri)"
  - "es_copy_file(in_uri, metaname, out_uri)"
  - "es_rename_file(in_uri, metaname, out_uri)"
  - "es_get_file_size(uri)"
tags:
  - component
  - cubrid
  - storage
  - lob
  - external-storage
related:
  - "[[components/storage|storage]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/object|object]]"
  - "[[components/transaction|transaction]]"
created: 2026-04-23
updated: 2026-04-23
---

# `es*.c` — External Storage (LOB Backends)

External storage handles the physical persistence of LOB (Large Object) values — BLOBs and CLOBs — outside the main database volumes. Each LOB is identified by a URI string that encodes the backend type and path.

## Architecture

```
SQL LOB column value
        │
        │  (stored as URI string in heap record)
        ▼
  LOB locator  (src/object/lob_locator.cpp)
        │
        │  es_create_file / es_write_file / es_read_file / es_delete_file
        ▼
      es.c  (unified dispatch layer)
        │
        ├──── ES_POSIX ──► es_posix.c ──► POSIX filesystem (hashed dirs)
        ├──── ES_OWFS  ──► es_owfs.c  ──► OWFS distributed filesystem
        └──── ES_LOCAL ──► es_posix.c ──► local read-only access
```

> [!info] LOB change = two-component edit
> The URI string lives in the heap record (managed by [[components/heap-file|heap_file.c]]). The physical bytes live on the filesystem managed by `es.c`. Changes to LOB behavior always span both files.

## URI Format

Each backend has a fixed prefix that doubles as its type discriminator:

| `ES_TYPE` | Prefix | Example |
|-----------|--------|---------|
| `ES_OWFS` | `owfs:` | `owfs:/cubrid/db1/lobs/ab/cdef...` |
| `ES_POSIX` | `file:` | `file:/var/cubrid/lobs/ab/cd/uuid` |
| `ES_LOCAL` | `local:` | `local:/path/to/file` |

`es_get_type(uri)` parses the prefix to determine which backend to dispatch to.

## POSIX Backend (`es_posix.c`)

Files are stored in a two-level hashed directory to avoid single-directory inode limits:

```
es_base_dir/
  ab/          ← first level: hash byte 1
    cd/        ← second level: hash byte 2
      <uuid>   ← actual LOB file
```

Hash constants: `ES_POSIX_HASH1 = 769`, `ES_POSIX_HASH2 = 381`.

`es_make_dirs` creates the two-level directory if it does not exist.

Key POSIX functions:
- `xes_posix_create_file(new_path)` — generate UUID-based filename, create directories
- `xes_posix_write_file(path, buf, count, offset)` — `pwrite` at offset
- `xes_posix_read_file(path, buf, count, offset)` — `pread` at offset
- `xes_posix_delete_file(path)` — `unlink`
- `xes_posix_copy_file(src_path, metaname, new_path)` — full file copy + rename

The `es_local_read_file` / `es_local_get_file_size` functions handle the `ES_LOCAL` type (read-only, used for importing files into the database).

## Initialization

`es_init(uri)` is called at server boot:
1. Parses the URI prefix to set the global `ES_TYPE`.
2. Calls the backend-specific init (`es_posix_init(base_path)` or OWFS equivalent).
3. After `es_init`, all `es_*` calls dispatch to the active backend.

`es_final()` shuts down the backend.

## Copy and Rename Semantics

- `es_copy_file(in_uri, metaname, out_uri)` — creates a new file with a new URI (used for `COPY` operations on rows that contain LOBs).
- `es_rename_file(in_uri, metaname, out_uri)` — rename within same backend (used on DML that changes the LOB's association).
- `es_move_file_with_prefix` — move with a new URI prefix (used for backup/restore operations).

`metaname` is the class/attribute name used for directory organization on some backends.

## Heap Integration

`heap_attrinfo_delete_lob` detects LOB-typed attribute values and calls `es_delete_file` for each. This is called on:
- `heap_delete_logical` (delete the row)
- `heap_update_logical` (replace the value, delete the old LOB)

> [!key-insight] LOB delete is not transactional at the filesystem level
> `es_delete_file` is a direct filesystem call with no WAL logging. If the transaction rolls back after calling it, the LOB file is already gone. CUBRID relies on a postpone-operation pattern: LOB physical deletion is deferred to after commit using a `LOG_POSTPONE` record appended to the log. Recovery replays the postpone on restart.

## Debugging

`es_log(...)` — controlled by `PRM_ID_DEBUG_ES` parameter; logs all external storage operations at debug level.

`es_get_unique_num()` generates unique filenames using a server-wide atomic counter.

## Related

- Parent: [[components/storage|storage]]
- [[components/heap-file]] — calls `es_delete_file` via `heap_attrinfo_delete_lob`
- [[components/object|object]] — LOB locator (`lob_locator.cpp`) manages URI lifecycle
- [[components/transaction]] — postpone log for LOB deletion ordering
