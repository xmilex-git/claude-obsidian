---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/overflow_file.c"
status: active
purpose: "Linked-page overflow storage for records too large for a single slotted-page slot; used by both heap_file (REC_BIGONE) and btree (long key overflow)"
key_files:
  - "overflow_file.c — implementation"
  - "overflow_file.h — public API, OVERFLOW_FIRST_PART / OVERFLOW_REST_PART structs"
public_api:
  - "overflow_insert(thread_p, ovf_vfid, ovf_vpid, recdes, file_type)"
  - "overflow_update(thread_p, ovf_vfid, ovf_vpid, recdes, file_type)"
  - "overflow_delete(thread_p, ovf_vfid, ovf_vpid)"
  - "overflow_get(thread_p, ovf_vpid, recdes, mvcc_snapshot)"
  - "overflow_get_nbytes(thread_p, ovf_vpid, recdes, start_offset, max_nbytes, remaining_length, mvcc_snapshot)"
  - "overflow_get_length(thread_p, ovf_vpid)"
  - "overflow_flush(thread_p, ovf_vpid)"
tags:
  - component
  - cubrid
  - storage
  - overflow
related:
  - "[[components/storage|storage]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/btree|btree]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/file-manager|file-manager]]"
created: 2026-04-23
updated: 2026-04-23
---

# `overflow_file.c` — Overflow Page Manager

Overflow files store records that exceed the capacity of a single slotted page. The overflow chain is a singly-linked list of pages, each pointing to the next via `next_vpid`.

## Page Layout

```c
/* First page of an overflow chain */
struct overflow_first_part {
  VPID next_vpid;    /* next page, or NULL if last */
  int  length;       /* total record length */
  char data[1];      /* record bytes start here */
};

/* Subsequent pages */
struct overflow_rest_part {
  VPID next_vpid;
  char data[1];
};
```

Each page stores as many bytes as fit after the header. `overflow_insert` allocates pages from `ovf_vfid` (the overflow `VFID` associated with the file) until the full record is written.

## Callers

| Caller | When |
|--------|------|
| `heap_file.c` | `REC_BIGONE` — record too large for a single heap page slot |
| `btree.c` | `BTREE_OVERFLOW_KEY` — key value too large for a leaf slot |

The overflow `VFID` for a heap file is found via `heap_ovf_find_vfid`. The btree overflow file VFID is stored in `BTID_INT.ovfid`.

## MVCC Support

`overflow_get` accepts an `MVCC_SNAPSHOT*`. For heap overflow records, the MVCC header is stored at the start of the first overflow page and is accessed separately via `heap_get_mvcc_rec_header_from_overflow`. The overflow chain itself does not have slot-level MVCC — each chain corresponds to one physical record version.

## Recovery Functions

| Function | Purpose |
|----------|---------|
| `overflow_rv_newpage_insert_redo` | Redo: initialize a new overflow page |
| `overflow_rv_newpage_link_undo` | Undo: unlink a newly added overflow page |
| `overflow_rv_link` | Undo/redo: update `next_vpid` link |
| `overflow_rv_page_update_redo` | Redo: apply data update to overflow page |

## Partial Read

`overflow_get_nbytes` allows reading a byte range from the overflow chain — used when only a prefix of a large value is needed (e.g. for `LIKE` prefix matching on `BLOB`/`CLOB` data).

## Related

- Parent: [[components/storage|storage]]
- [[components/heap-file]] — primary caller for `REC_BIGONE` records
- [[components/btree]] — caller for long B-tree key overflow
- [[components/page-buffer]] — all overflow pages are fixed/unfixed through the buffer pool
- [[components/file-manager]] — allocates overflow pages via `file_alloc`
