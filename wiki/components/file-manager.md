---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/"
status: active
purpose: "Three-layer storage stack: file_manager allocates pages within virtual files; disk_manager allocates sectors within volumes; file_io performs raw pread/pwrite with optional TDE encryption and backup"
key_files:
  - "file_manager.c / file_manager.h — page and file allocation, FILE_TYPE catalog"
  - "disk_manager.c / disk_manager.h — sector bitmap, volume add/remove, VSID allocation"
  - "file_io.c / file_io.h — raw I/O, volume descriptor table, backup, TDE flag"
public_api:
  - "file_alloc(thread_p, vfid, init_page_func, init_page_args, vpid_out, page_out)"
  - "file_dealloc(thread_p, vfid, vpid, file_type)"
  - "file_create(thread_p, file_type, size_hint_npages, des, class_oid, vpid_out, is_new_file)"
  - "file_destroy(thread_p, vfid, is_temp)"
  - "file_get_num_data_sectors(thread_p, vfid, *n_sectors_out) — assignment-style read of n_sector_full + n_sector_partial (added by PR #7011, merge cc563c7f)"
  - "disk_reserve_sectors(thread_p, purpose, volid_hint, n_sectors, reserved_sectors)"
  - "disk_unreserve_ordered_sectors(thread_p, purpose, n_sectors, sectors)"
  - "disk_add_volume_extension(thread_p, purpose, voltype, npages, path, name, ...)"
  - "fileio_open(vlabel, flags, mode) -> int (fd)"
  - "fileio_read(thread_p, vdes, io_page, pageid, pagesize)"
  - "fileio_write(thread_p, vdes, io_page, pageid, pagesize, write_mode)"
  - "fileio_sync(thread_p, vdes)"
  - "fileio_backup / fileio_restore"
tags:
  - component
  - cubrid
  - storage
  - file-manager
  - disk-manager
  - file-io
related:
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/double-write-buffer|double-write-buffer]]"
  - "[[components/transaction|transaction]]"
created: 2026-04-23
updated: 2026-04-23
---

# File Manager / Disk Manager / File I/O

These three layers form the storage stack beneath the page buffer:

```
file_manager.c       ← logical: "give me a new page in this file"
      │
      ▼
disk_manager.c       ← physical: "reserve N sectors on volume V"
      │
      ▼
file_io.c            ← raw: pread/pwrite on a volume file descriptor
```

## Layer 1: File Manager (`file_manager.c`)

### File Types

Defined in `file_manager.h` as `FILE_TYPE`:

| `FILE_TYPE` | Role |
|-------------|------|
| `FILE_TRACKER` | Global file tracker — maps all files in all volumes |
| `FILE_HEAP` | Heap (class instances) |
| `FILE_HEAP_REUSE_SLOTS` | Heap with OID reuse after delete |
| `FILE_MULTIPAGE_OBJECT_HEAP` | Overflow heap for large objects |
| `FILE_BTREE` | B-tree index |
| `FILE_BTREE_OVERFLOW_KEY` | Overflow storage for long keys |
| `FILE_EXTENDIBLE_HASH` | EH bucket pages |
| `FILE_EXTENDIBLE_HASH_DIRECTORY` | EH directory pages |
| `FILE_CATALOG` | System catalog |
| `FILE_DROPPED_FILES` | Tracks files dropped but not yet freed |
| `FILE_VACUUM_DATA` | Vacuum tracking pages |
| `FILE_QUERY_AREA` | Temporary storage for query results (list files) |
| `FILE_TEMP` | General temp files (sort, hash join) |

### File Descriptors

File descriptors store type-specific metadata alongside the VFID:

| Descriptor type | Used by |
|-----------------|---------|
| `FILE_HEAP_DES` — `{class_oid, hfid}` | Heap files |
| `FILE_OVF_HEAP_DES` — `{hfid, class_oid}` | Overflow heap files |
| `FILE_BTREE_DES` — `{class_oid, attr_id}` | B-tree files |

### Page Allocation

`file_alloc` allocates a new page within an existing file:
1. Looks up the file's free page list in the file tracker.
2. If empty, asks `disk_manager` for more sectors.
3. Returns the new `VPID` and optionally calls `init_page_func` with the new page fixed.

`file_dealloc` returns a page to the free list, logging the operation for recovery.

> [!key-insight] Two-phase deallocation for MVCC
> When a file page is deallocated under MVCC, it may still be visible to older snapshots. File manager uses `FILE_DROPPED_FILES` tracking and a postpone-operation pattern to defer physical reclamation until after all concurrent transactions that might see the page have committed.

### Data-sector harvesting (`file_get_all_data_sectors`)

`int file_get_all_data_sectors(THREAD_ENTRY *, VFID *, FILE_FTAB_COLLECTOR *)` walks the file's allocation metadata and emits every data-bearing sector into the collector.

- `FILE_FTAB_COLLECTOR` accumulates `(vsid, bitmap)` entries — one per allocated sector — where the bitmap marks which of the 64 pages inside that sector actually hold data.
- The walk covers both `PART_FTAB` (partial sectors — sectors with mixed free + allocated pages) and `FULL_FTAB` (full sectors — all 64 pages allocated).
- The argument type is `VFID *` — the helper is file-type agnostic (callable for heap files, temp/list files, etc.). Earlier draft used `HFID *`; widened during review (`@hornetmj`) so the same collector works for non-heap files like `QMGR_TEMP_FILE`.
- Used by: parallel heap scan (sector pre-split of a table's heap), parallel list scan (sector pre-split of a temp file plus its dependent chain), parallel index build (`SORT_INDEX_LEAF`, via PR #7011), and **parallel hash join split phase** (via PR #6981 — wrapped by `qfile_collect_list_sector_info` in [[components/list-file]] which walks the `dependent_list_id` chain and concatenates sectors with a parallel `tfiles[]` array).
- Thread safety: the collector lives on the caller's frame; `file_get_all_data_sectors` is called single-threaded on the main thread at scan open, before workers start.

> [!update] PR #6981 (merge `0be6cdf6`) — new consumer
> Parallel hash join's split phase now uses `file_get_all_data_sectors` (indirectly, via `qfile_collect_list_sector_info`) to pre-split input list-file pages across workers, replacing the previous `scan_mutex`-serialised page handoff. Same primitive as parallel heap scan (PR #6911); each worker claims a sector via `next_sector_index.fetch_add(1)` then walks the bitmap with `__builtin_ctzll`.

The `FILE_FULL_PAGE_BITMAP` macros in `file_manager.h` encode the per-sector 64-page presence vector as a `uint64_t`.

### Sector-count helper (`file_get_num_data_sectors`)

> [!update] PR #7011 (merge `cc563c7f`) — new helper
> ```c
> int file_get_num_data_sectors (THREAD_ENTRY *thread_p, const VFID *vfid, int *n_sectors_out);
> ```
> Reads `*n_sectors_out = fhead->n_sector_full + fhead->n_sector_partial;` — **direct assignment, not accumulation** (callers do not need to zero `*n_sectors_out` first, but a greptile review thread on PR #7011 misread this as accumulation; the assignment semantics are intentional). Used by `external_sort.c::sort_check_parallelism` for `SORT_INDEX_LEAF` to decide whether parallel CREATE INDEX is worthwhile by comparing the heap file's sector count against a threshold.

## Layer 2: Disk Manager (`disk_manager.c`)

### Volume Model

Each volume is a flat file on the OS filesystem. Multiple volumes can exist per database:

```
volume 0 (permanent)  ── sectors ──► pages
volume 1 (expansion)  ── sectors ──► pages
...
temp volume           ── sectors ──► pages
```

### Sector Allocation

The disk manager tracks free sectors with a bitmap per volume. The `DISK_VOLUME_SPACE_INFO` struct tracks:
- `n_max_sects` — total capacity
- `n_total_sects` — reserved sectors
- `n_free_sects` — free sectors

`disk_reserve_sectors(thread_p, purpose, volid_hint, n_sectors, reserved_sectors)` atomically reserves N sectors, choosing a volume via `volid_hint` or auto-selecting. Returns an array of `VSID` (volume-sector identifiers).

`disk_unreserve_ordered_sectors` releases sectors back to free.

`DISK_SECTS_NPAGES(nsects)` converts sectors to pages; `DISK_PAGES_TO_SECTS(npages)` is the ceiling inverse.

### Volume Operations

| Function | Role |
|----------|------|
| `disk_format_first_volume` | Initialize the first volume at database creation |
| `disk_add_volume_extension` | Add a new volume (e.g. `ALTER DATABASE ADD VOLUME`) |
| `disk_unformat` | Remove a volume |
| `disk_set_checkpoint` | Update checkpoint LSA in volume header |
| `disk_is_page_sector_reserved` | Debug validation that a page is in a reserved sector |

`disk_lock_extend` / `disk_unlock_extend` serialize volume addition against concurrent disk operations.

## Layer 3: File I/O (`file_io.c`)

### Volume Descriptor Table

`file_io.c` maintains an open file descriptor (`int vdes`) for each volume. Volumes are opened at boot (`fileio_open`) and closed at shutdown.

`NULL_VOLDES` (-1) signals an invalid/closed volume.

### Read / Write

```c
/* Read one page from disk */
fileio_read (thread_p, vdes, io_page, pageid, pagesize);

/* Write one page to disk */
fileio_write (thread_p, vdes, io_page, pageid, pagesize, FILEIO_WRITE_DEFAULT_WRITE);
```

`io_page` is a `FILEIO_PAGE*` — it includes the page header (`FILEIO_PAGE_RESERVED`) before the data bytes.

### TDE Encryption

`FILEIO_PAGE_RESERVED` carries `pflag`:
- `FILEIO_PAGE_FLAG_ENCRYPTED_AES` (0x1)
- `FILEIO_PAGE_FLAG_ENCRYPTED_ARIA` (0x2)

When `pgbuf_set_tde_algorithm` marks a BCB for encryption, `file_io.c` encrypts the page data on write and decrypts on read. The key is managed by the TDE module (`tde.h`).

### Backup / Restore

`fileio_backup` reads all volumes and writes to a backup medium (file or device). Incremental backups use page LSAs to identify changed pages. `fileio_restore` applies a full backup and incremental volumes in order.

### Sync

`fileio_sync(thread_p, vdes)` calls `fsync` on the volume. Called by the [[components/double-write-buffer|double-write buffer]] after writing a DWB block and by `pgbuf_flush_all`.

## Latch-to-Sector Ordering

> [!warning] Page allocation requires write latch on the file tracker
> `file_alloc` acquires a write latch on the file tracker page. This must happen before latching any data page in the same operation. Violating this order can cause deadlock with concurrent creates.

## Related

- Parent: [[components/storage|storage]]
- [[components/page-buffer]] — calls `fileio_read/write` through the buffer pool flush path
- [[components/double-write-buffer]] — sits between buffer pool flush and `fileio_write`
- [[components/transaction]] — log is synced before pages flush (WAL)
