---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/"
status: active
purpose: "On-disk and in-memory storage primitives: buffer pool, heap files, B-tree indexes, file/disk management, external sort, extendible hashing, double-write buffer, and LOB external storage"
key_files:
  - "page_buffer.c / page_buffer.h (buffer pool: LRU zones, fix/unfix, dirty tracking)"
  - "heap_file.c / heap_file.h (row-oriented storage, MVCC-aware scan)"
  - "btree.c / btree.h (B+tree: find, range scan, insert, delete, MVCC ops)"
  - "btree_load.c / btree_load.h (bulk index loading)"
  - "btree_unique.hpp (unique constraint stats: btree_unique_stats, multi_index_unique_stats)"
  - "file_manager.c / file_manager.h (page allocation, file types)"
  - "file_io.c / file_io.h (raw volume I/O, backup, TDE encryption flag)"
  - "disk_manager.c / disk_manager.h (sector allocation, volume management)"
  - "double_write_buffer.hpp / double_write_buffer.c (torn-write protection)"
  - "overflow_file.c / overflow_file.h (large-record overflow pages)"
  - "extendible_hash.c / extendible_hash.h (disk-based hash, internal use)"
  - "external_sort.c / external_sort.h (merge sort over temp files)"
  - "slotted_page.c / slotted_page.h (slot-directory page layout)"
  - "es.c / es.h (external storage API for LOBs)"
  - "es_posix.c / es_posix.h (POSIX filesystem LOB backend)"
  - "catalog_class.h (catalog heap operations)"
  - "oid.c / oid.h (OID utilities)"
  - "storage_common.h (VPID, OID, HFID, BTID, VSID — core identifiers)"
public_api:
  - "pgbuf_fix(thread_p, vpid, fetch_mode, latch_mode, condition)"
  - "pgbuf_unfix(thread_p, pgptr)"
  - "pgbuf_set_dirty(thread_p, pgptr, free_page)"
  - "heap_insert_logical / heap_delete_logical / heap_update_logical"
  - "heap_next / heap_first / heap_last (sequential scan)"
  - "btree_insert / btree_mvcc_delete / btree_update"
  - "btree_keyval_search / btree_range_scan"
  - "sort_listfile(...) (external sort entry point)"
  - "es_create_file / es_write_file / es_read_file / es_delete_file"
  - "overflow_insert / overflow_get / overflow_delete"
  - "dwb_add_page / dwb_flush_force"
tags:
  - component
  - cubrid
  - storage
  - server
related:
  - "[[modules/src|src]]"
  - "[[components/transaction|transaction]]"
  - "[[components/object|object]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/parallel-sort|parallel-sort]]"
  - "[[Architecture Overview]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/storage/` — Storage Layer

The storage layer is the lowest server-side tier that all other components depend on. It owns every byte on disk: how pages are cached in memory, how records are arranged inside pages, how B-tree indexes are maintained, and how large objects are stored outside the main database volumes. It is available in `SERVER_MODE` and `SA_MODE` only.

## Architecture Overview

```
         ┌──────────────────────────────────────────────────────┐
         │              Callers (heap, btree, sort, …)          │
         └──────────────┬───────────────────────────────────────┘
                        │  pgbuf_fix / pgbuf_unfix
         ┌──────────────▼───────────────────────────────────────┐
         │            Page Buffer Pool  (page_buffer.c)         │
         │   BCB hash table  |  LRU zones 1/2/3  |  dirty list │
         └───────┬──────────────────────────┬───────────────────┘
                 │ pgbuf_flush_with_wal      │ victim flush
         ┌───────▼───────────┐      ┌───────▼───────────────────┐
         │ Double-Write Buf  │      │    File I/O (file_io.c)   │
         │ (dwb_add_page)    │      │  raw pread/pwrite, TDE    │
         └───────┬───────────┘      └───────────────────────────┘
                 │
         ┌───────▼───────────────────────────────────────────────┐
         │           Disk Manager  (disk_manager.c)              │
         │  sector bitmap  |  volume add/remove  |  VSID alloc  │
         └───────────────────────────────────────────────────────┘

Higher-level sub-systems:
  Heap File ──── Slotted Page ──── Page Buffer
  B-tree   ──────────────────────── Page Buffer
  Overflow File ─────────────────── Page Buffer
  External Sort ─────── temp file ── File I/O
  Extendible Hash ───── page buffer ─ Page Buffer
  External Storage (LOB) ─── POSIX/OWFS filesystem (bypasses page buffer)
```

## Sub-Systems

| Sub-system | File(s) | Wiki page |
|------------|---------|-----------|
| Buffer pool | `page_buffer.c/h` | [[components/page-buffer]] |
| B-tree index | `btree.c/h`, `btree_load.c/h`, `btree_unique.hpp` | [[components/btree]] |
| Heap file | `heap_file.c/h`, `heap_attrinfo.c/h` | [[components/heap-file]] |
| File + disk mgr | `file_manager.c/h`, `file_io.c/h`, `disk_manager.c/h` | [[components/file-manager]] |
| Double-write buf | `double_write_buffer.hpp/.c` | [[components/double-write-buffer]] |
| Overflow file | `overflow_file.c/h` | [[components/overflow-file]] |
| Extendible hash | `extendible_hash.c/h` | [[components/extendible-hash]] |
| External sort | `external_sort.c/h` | [[components/external-sort]] |
| External storage | `es.c/h`, `es_posix.c/h`, `es_owfs.c/h`, `es_common.h` | [[components/external-storage]] |
| Slotted page | `slotted_page.c/h` | (inline with heap-file page) |

## Core Identifiers (`storage_common.h`)

| Type | Fields | Purpose |
|------|--------|---------|
| `VPID` | `volid`, `pageid` | Volume + page address — smallest addressable unit |
| `VSID` | `volid`, `sectid` | Volume + sector (disk allocation unit) |
| `OID` | `volid`, `pageid`, `slotid` | Object identifier — physical row address |
| `VFID` | `volid`, `fileid` | Virtual file identifier |
| `HFID` | `vfid`, `hpgid` | Heap file identifier |
| `BTID` | `vfid`, `root_pageid` | B-tree identifier |
| `EHID` | — | Extendible hash identifier |
| `LOG_LSN` | `pageid`, `offset` | Log sequence number (also `log_lsa`) |

> [!key-insight] OID is a physical address
> An `OID` directly encodes the disk location `(volid, pageid, slotid)`. There is no separate indirection table — changing a row's physical location (e.g. during update or vacuum) requires updating every index that points to the old OID.

## Buffer Pool Protocol

```c
/* Fix (pin) a page — MUST be matched by unfix */
PAGE_PTR page = pgbuf_fix (thread_p, &vpid, OLD_PAGE, PGBUF_LATCH_READ,
                            PGBUF_UNCONDITIONAL_LATCH);
/* ... read or modify page ... */
pgbuf_set_dirty (thread_p, page, DONT_FREE);   /* if modified */
pgbuf_unfix (thread_p, page);                   /* always */
```

- Every `pgbuf_fix()` must have exactly one matching `pgbuf_unfix()`. Tracked by `resource_tracker` in debug builds.
- Latch modes: `PGBUF_LATCH_READ` (shared), `PGBUF_LATCH_WRITE` (exclusive), `PGBUF_LATCH_FLUSH` (block mode only, not fixable).
- Latch conditions: `PGBUF_UNCONDITIONAL_LATCH` (block until granted), `PGBUF_CONDITIONAL_LATCH` (fail immediately if busy).
- Page types: `PAGE_HEAP`, `PAGE_BTREE`, `PAGE_OVERFLOW`, `PAGE_CATALOG`, `PAGE_VACUUM_DATA`, etc.

> [!warning] Latch vs lock distinction
> Page **latches** (short-term, physical consistency) are distinct from transaction **locks** (logical, MVCC-based). B-tree operations hold page latches but acquire transaction locks separately through [[components/transaction|lock_manager]]. Mixing up the two is a common source of deadlock bugs.

## Page Types (`FILE_TYPE` enum)

Defined in `file_manager.h`:

| `FILE_TYPE` | Purpose |
|-------------|---------|
| `FILE_HEAP` | Normal heap (class instances) |
| `FILE_HEAP_REUSE_SLOTS` | Heap that reuses slot IDs after delete |
| `FILE_BTREE` | B-tree index pages |
| `FILE_BTREE_OVERFLOW_KEY` | Overflow storage for long keys |
| `FILE_EXTENDIBLE_HASH` | Extendible hash bucket pages |
| `FILE_EXTENDIBLE_HASH_DIRECTORY` | EH directory pages |
| `FILE_CATALOG` | System catalog |
| `FILE_TEMP` | Temporary files (sort, hash join) |
| `FILE_VACUUM_DATA` | Vacuum tracking data |
| `FILE_TRACKER` | File tracker (maps volume files) |

## Cross-Cutting Concerns

### WAL Coordination with Transaction Layer

All page modifications must follow the Write-Ahead Logging protocol:
1. Append a log record via `log_append*` before (or at) `pgbuf_set_dirty`.
2. On flush, `pgbuf_flush_with_wal` ensures the log record is synced before the page.
3. Checkpoint uses `pgbuf_flush_checkpoint` to flush pages up to a given LSN.
4. Each page carries an embedded `LOG_LSA` (page LSA), set via `pgbuf_set_lsa`.

> [!key-insight] WAL ordering enforced through page LSA
> The buffer pool checks the page's current LSA against the log at flush time. A page cannot be written to disk unless its LSA is already present in the durable log. This is enforced inside `pgbuf_flush_with_wal`, which calls `log_flush_if_needed`.

### LOB Cross-Cutting with Object Layer

LOB data crosses two components:
- **LOB locator** lives in [[components/object|src/object/lob_locator.cpp]].
- **Physical storage** is handled here by `es.c`, routing to a POSIX or OWFS backend.
- Heap operations (`heap_attrinfo_delete_lob`) bridge the two: they detect LOB attributes and call `es_delete_file`.

> [!info] LOB change = multi-file edit
> Any behavioral change to LOB handling (e.g. new backend, different naming) requires changes in both `src/storage/es*.c` and `src/object/lob_locator.cpp`.

### TDE Encryption

Pages can be encrypted at the I/O layer. `FILEIO_PAGE_FLAG_ENCRYPTED_AES` and `_ARIA` bits are stored in the page header's `pflag`. `pgbuf_set_tde_algorithm` marks a page for encryption; `file_io.c` performs the actual encrypt/decrypt on read and write.

### Parallel Sort Interface

`external_sort.c` exposes `sort_listfile()`, which is consumed by [[components/parallel-sort|parallel-sort]] through macros `SORT_EXECUTE_PARALLEL` / `SORT_WAIT_PARALLEL`. The external sort subsystem is also called directly by the non-parallel query executor.

> [!update] PR #7011 (merge `cc563c7f`) — `external_sort.c` and `btree_load.c` now consume `parallel_query::ftab_set`
> The `ftab_set` value type (per-worker `FILE_PARTIAL_SECTOR` slice) was promoted from `parallel_heap_scan` to the umbrella `parallel_query` namespace and its header relocated to `src/query/parallel/px_ftab_set.hpp`. `external_sort.c::sort_start_parallelism` (SORT_INDEX_LEAF arm) and `btree_load.c::btree_sort_get_next_parallel` consume it directly to drive the per-worker heap-sector iteration during parallel CREATE INDEX. See [[components/parallel-heap-scan-input-handler#ftab-set-header-only-px-ftab-set-hpp|ftab_set details]].

## Common Bug Locations

| Symptom | File | Entry point |
|---------|------|-------------|
| Wrong query result from index | `btree.c` | `btree_range_scan`, `btree_keyval_search` |
| Heap record corruption | `slotted_page.c` | slot directory logic |
| Buffer leak (fixed but not unfixed) | `page_buffer.c` | `resource_tracker` dump in debug |
| Disk space not reclaimed | `disk_manager.c`, `file_manager.c` | sector/page deallocation |
| Overflow record wrong content | `overflow_file.c` | `overflow_get`, `overflow_update` |
| Torn page after crash | `double_write_buffer.c` | `dwb_load_and_recover_pages` |
| LOB missing after delete | `es.c`, `heap_file.c` | `heap_attrinfo_delete_lob`, `es_delete_file` |

## Related

- Parent: [[modules/src|src]]
- [[components/transaction]] — WAL, lock manager, MVCC visibility
- [[components/object|object]] — LOB locator, class representation
- [[components/parallel-query|parallel-query]] — parallel heap scan accesses heap + page buffer
- [[components/parallel-sort|parallel-sort]] — calls `sort_listfile` from external_sort
- [[Memory Management Conventions]] — `db_private_alloc`, `free_and_init`
- [[Build Modes (SERVER SA CS)]] — storage files guarded by `SERVER_MODE` / `SA_MODE`
- Source: [[sources/cubrid-src-storage]]
