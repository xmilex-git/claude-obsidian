---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/double_write_buffer.hpp"
status: active
purpose: "Torn-write protection: buffer dirty pages into a separate sequential DWB file before writing to their actual volume locations, so a crash mid-write leaves a consistent copy to recover from"
key_files:
  - "double_write_buffer.hpp — public API and DWB_SLOT struct"
  - "double_write_buffer.c — full implementation"
  - "file_io.h / file_io.c — underlying sync calls"
public_api:
  - "dwb_create(thread_p, dwb_path_p, db_name_p)"
  - "dwb_destroy(thread_p)"
  - "dwb_add_page(thread_p, io_page_p, vpid, ensure_metadata, p_dwb_slot)"
  - "dwb_set_data_on_next_slot(thread_p, io_page_p, can_wait, ensure_metadata, p_dwb_slot)"
  - "dwb_flush_force(thread_p, all_sync)"
  - "dwb_read_page(thread_p, vpid, io_page, success)"
  - "dwb_load_and_recover_pages(thread_p, dwb_path_p, db_name_p)"
  - "dwb_synchronize(thread_p, vol_fd, vlabel)"
tags:
  - component
  - cubrid
  - storage
  - crash-safety
  - double-write
related:
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/file-manager|file-manager]]"
created: 2026-04-23
updated: 2026-04-23
---

# `double_write_buffer` — Torn-Write Protection

The double-write buffer (DWB) prevents torn-page corruption when the OS or hardware crashes mid-write. A partial write of a 16 KB page to a database volume could leave half-old, half-new data that WAL cannot recover (WAL only handles logical changes, not partial physical writes).

## How It Works

```
pgbuf_flush_with_wal()
    │
    ├─ 1. dwb_add_page()          ← copy page to DWB slot
    │       │
    │       ▼
    │   DWB_BLOCK (e.g. 2 MB)
    │   accumulate slots until block is full or flush forced
    │       │
    │       ├─ 2. fileio_sync(DWB file)   ← fsync DWB
    │       │
    │       └─ 3. fileio_write(volume, page)  ← write to actual location
    │               │
    │               └─ 4. fileio_sync(volume)  ← optional: sync volume
    │
    └─ return to buffer pool
```

If a crash occurs at step 3, the DWB still has the complete page. Recovery calls `dwb_load_and_recover_pages`, which re-applies all valid slots from the DWB to their actual volume locations before WAL recovery begins.

## DWB_SLOT

```c
struct double_write_slot {
  FILEIO_PAGE *io_page;         /* full page data (including page header) */
  VPID         vpid;            /* destination volume + page id */
  LOG_LSA      lsa;             /* page LSA at the time of write */
  bool         ensure_metadata; /* include metadata (LSA, ptype) in sync */
  unsigned int position_in_block;
  unsigned int block_no;
};
```

Slots are arranged in fixed-size blocks (configurable). A block is flushed when full or when `dwb_flush_force` is called.

## Recovery

`dwb_load_and_recover_pages` is called early in server startup (before WAL redo):
1. Opens the DWB file and reads all valid slots.
2. For each slot, compares the slot's LSA to the current page LSA on disk.
3. If the DWB slot has a higher (more recent) LSA, re-writes the page from the DWB to the volume.
4. This ensures no partial writes survive into the WAL recovery phase.

> [!key-insight] DWB recovery precedes WAL redo
> The DWB recovery pass runs before `log_recovery` applies redo records. This guarantees the on-disk page state is physically consistent before logical redo begins. Without this ordering, redo could produce incorrect results on a partially-written page.

## Configuration

| Parameter | Role |
|-----------|------|
| `PRM_ID_DWB_SIZE` | Total DWB file size in pages |
| `PRM_ID_DWB_BLOCKS` | Number of blocks in the DWB |

DWB is created at server start via `dwb_create`, destroyed via `dwb_destroy`. `dwb_recreate` handles size/count changes.

## Read Path

`dwb_read_page` allows reading a page from the DWB if it is present and more recent than the on-disk version. This is used during recovery to avoid reading a stale (partially written) page from the volume.

## Server Mode Daemons

In `SERVER_MODE`, two daemons are initialized by `dwb_daemons_init`:
- A flush daemon that periodically calls `dwb_flush_force`.
- A block consumer that asynchronously writes completed blocks to volumes.

## Related

- Parent: [[components/storage|storage]]
- [[components/page-buffer]] — calls `dwb_add_page` before each page write
- [[components/file-manager]] — `fileio_sync` is the final step after DWB write
