---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/heap_file.c"
status: active
purpose: "Row-oriented heap file storage: record insert/update/delete with MVCC, sequential and range scans, class representation cache, overflow record management, and statistics maintenance"
key_files:
  - "heap_file.c (~27K lines) — full implementation"
  - "heap_file.h — HEAP_SCANCACHE, HEAP_OPERATION_CONTEXT, HEAP_GET_CONTEXT, scan/modify APIs"
  - "heap_attrinfo.c / heap_attrinfo.h — attribute cache, key generation for indexes"
  - "slotted_page.c / slotted_page.h — slot directory + record layout within pages"
public_api:
  - "heap_insert_logical(thread_p, context, home_hint_p)"
  - "heap_delete_logical(thread_p, context)"
  - "heap_update_logical(thread_p, context)"
  - "heap_next(thread_p, hfid, class_oid, next_oid, recdes, scan_cache, ispeeking)"
  - "heap_first / heap_last / heap_prev"
  - "heap_get_visible_version(thread_p, oid, class_oid, recdes, scan_cache, ispeeking, old_chn)"
  - "heap_scancache_start / heap_scancache_end"
  - "heap_scancache_start_modify / heap_scancache_end_modify"
  - "heap_classrepr_get / heap_classrepr_free"
  - "heap_attrinfo_start / heap_attrinfo_read_dbvalues / heap_attrinfo_end"
  - "heap_attrinfo_generate_key(...) — derive index key from attributes"
  - "heap_assign_address(thread_p, hfid, class_oid, oid, expected_length)"
  - "heap_get_class_oid / heap_get_class_name"
  - "heap_ovf_find_vfid / heap_ovf_delete"
tags:
  - component
  - cubrid
  - storage
  - heap
  - mvcc
related:
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/btree|btree]]"
  - "[[components/overflow-file|overflow-file]]"
  - "[[components/transaction|transaction]]"
  - "[[components/external-storage|external-storage]]"
created: 2026-04-23
updated: 2026-04-23
---

# `heap_file.c` — Heap File Manager

The heap file is CUBRID's primary on-disk format for storing class instances (rows). Each class maps to one `HFID` (heap file identifier). Pages within a heap use the slotted-page format (`slotted_page.c`) where slot 0 is the page header/chain record.

## Page Layout (Slotted Page)

```
┌─────────────────────────────────────────────────┐
│ Page header (slot 0: HEAP chain + stats)         │
│ Record 1  │  Record 2  │ ... │  Record N         │
│                             │                    │
│ ...free space...            │                    │
│                             │                    │
│ Slot N │ ... │ Slot 2 │ Slot 1 │ Slot dir hdr   │
└─────────────────────────────────────────────────┘
  (records grow from low address, slots from high address)
```

`HEAP_DROP_FREE_SPACE` = 30% of page size. Pages with less than this threshold are dropped from the best-space list and will not accept new inserts until compacted.

## Record Types

| REC_TYPE | Meaning |
|----------|---------|
| `REC_HOME` | Normal record fully in this page |
| `REC_NEWHOME` | Relocated record — the new location after update |
| `REC_RELOCATION` | Forwarding record — contains OID of the new home |
| `REC_BIGONE` | First page of a multi-page (overflow) record |
| `REC_ASSIGN_ADDRESS` | Placeholder OID assigned before insert |
| `REC_MARKDELETED` | Slot is logically deleted (slot reuse pending) |

## MVCC Record Header

Every record in an MVCC-enabled class has a `MVCC_REC_HEADER` prepended:

| Field | Meaning |
|-------|---------|
| `mvcc_ins_id` | Transaction that inserted this version |
| `mvcc_del_id` | Transaction that deleted this version (`MVCCID_NULL` if live) |
| `mvcc_flag` | Flags: HAS_INSID, HAS_DELID, etc. |
| `repr_id` | Schema representation version |
| `chn` | Cache coherency number |
| `prev_version_lsa` | LSA of the previous version (for undo chain traversal) |

`mvcc_header_size_lookup[flags]` gives the actual byte size for the current flag combination.

## Vacuum Status Tracking

Each heap page carries a `HEAP_PAGE_VACUUM_STATUS`:

| Status | Meaning |
|--------|---------|
| `HEAP_PAGE_VACUUM_NONE` | No MVCC garbage — page is clean |
| `HEAP_PAGE_VACUUM_ONCE` | One vacuum pass needed |
| `HEAP_PAGE_VACUUM_UNKNOWN` | Multiple MVCC ops without vacuum — state unpredictable |

This allows heap pages to be safely deallocated when all MVCC garbage has been cleaned, without risking future vacuum worker access.

## Operation Context

All logical DML operations use `HEAP_OPERATION_CONTEXT`:

```c
struct heap_operation_context {
  HEAP_OPERATION_TYPE  type;         /* INSERT / DELETE / UPDATE */
  UPDATE_INPLACE_STYLE update_in_place;
  HFID                 hfid;
  OID                  oid, class_oid;
  RECDES              *recdes_p;
  HEAP_SCANCACHE      *scan_cache_p;
  /* page watchers for home, overflow, header, forward pages */
  PGBUF_WATCHER home_page_watcher, overflow_page_watcher,
                header_page_watcher, forward_page_watcher;
  OID   res_oid;          /* output: assigned OID */
  bool  is_logical_old;
  bool  is_bulk_op;       /* disables MVCC side effects for bulk insert */
  bool  use_bulk_logging;
  LOG_LSA supp_undo_lsa, supp_redo_lsa;  /* supplemental logging */
};
```

Create the appropriate context with:
- `heap_create_insert_context`
- `heap_create_delete_context`
- `heap_create_update_context`

## Scan Cache (`HEAP_SCANCACHE`)

The scan cache keeps the last-fixed page pinned between calls to `heap_next`:

```c
struct heap_scancache {
  HEAP_SCANCACHE_NODE  node;         /* current hfid + class_oid */
  LOCK                 page_latch;   /* NULL_LOCK when class has SIX/X lock */
  bool                 cache_last_fix_page;
  bool                 mvcc_disabled_class;
  PGBUF_WATCHER        page_watcher;
  MVCC_SNAPSHOT       *mvcc_snapshot;
  multi_index_unique_stats *m_index_stats;  /* per-op unique stats */
  /* private area for record data */
  cubmem::single_block_allocator *m_area;
};
```

Start/end pairs:
- `heap_scancache_start` / `heap_scancache_end` — for read scans
- `heap_scancache_start_modify` / `heap_scancache_end_modify` — for DML

## MVCC Visibility

> [!key-insight] Two visibility paths
> `heap_file.c` has two main visibility paths: the MVCC-aware path (`heap_get_visible_version`, `heap_scan_get_visible_version`) that applies the transaction's MVCC snapshot, and the non-MVCC path for MVCC-disabled classes (e.g. `_db_serial`). Always check `HEAP_SCANCACHE.mvcc_disabled_class` when debugging scan results.

MVCC delete path:
1. `heap_delete_logical` sets the delete MVCCID in the record header (marking it as a deleted version).
2. The old OID is still visible to snapshot-isolated readers.
3. Vacuum later calls `heap_vacuum_all_objects` to physically remove dead versions.
4. If the record spans overflow pages, each overflow page's MVCC header is also updated (`heap_set_mvcc_rec_header_on_overflow`).

## Best-Space List

The heap maintains a `HEAP_BESTSPACE` list: a small list of pages with the most free space, used to direct new inserts. `heap_stats_update` is called on every unfix of a modified heap page to update the estimate. `HEAP_DROP_FREE_SPACE` (30% of page) is the threshold below which a page is dropped from the best-space list.

## Class Representation Cache

`heap_classrepr_get` returns an `OR_CLASSREP` (object representation) — the schema for a class at a given representation version. Results are cached in a per-server hash table keyed by `class_oid + repr_id`. `heap_classrepr_decache` invalidates a class's cache entry on schema change.

The `HEAP_HFID_TABLE` (`heap_hfid_table`) is a lock-free hash table (`LF_HASH_TABLE`) mapping `class_oid → (HFID, FILE_TYPE, classname)`. This avoids repeated heap lookups for class metadata on every scan.

## Large Records (REC_BIGONE)

Records too large for a single page are stored as multi-page overflow sequences via [[components/overflow-file|overflow_file.c]]:
1. `heap_assign_address` assigns an OID with `REC_ASSIGN_ADDRESS`.
2. `heap_insert_logical` detects `is_big_length(length)` and calls `overflow_insert` to write overflow pages.
3. The heap slot contains a `REC_BIGONE` record with the VPID of the first overflow page.
4. On read, `heap_get_record_data_when_all_ready` follows the overflow chain.

## LOB Integration

`heap_attrinfo_delete_lob` iterates attributes in `HEAP_CACHE_ATTRINFO`, detects LOB-typed values, and calls `es_delete_file` for each LOB URI. This is the bridge between heap delete and [[components/external-storage|external storage]].

## Recovery Functions

| Function | Operation |
|----------|-----------|
| `heap_rv_undo_insert` | Undo a heap insert |
| `heap_rv_redo_insert` / `heap_rv_mvcc_redo_insert` | Redo insert (MVCC variant) |
| `heap_rv_undo_delete` / `heap_rv_redo_delete` | Undo/redo delete |
| `heap_rv_mvcc_undo_delete` / `heap_rv_mvcc_redo_delete_*` | MVCC delete variants |
| `heap_rv_undo_update` / `heap_rv_redo_update` | Update |
| `heap_rv_update_chain_after_mvcc_op` | Update prev-version LSA chain after MVCC op |

## Sampling Scan Integration

`heap_next_internal` branches on whether the caller supplied a `HEAP_SAMPLING_INFO *sampling` in the scan context. When set, after each yielded record the scan advances by `sampling->weight` pages via `heap_vpid_skip_next` — i.e. uniform-stride page sampling. Weight is computed upstream in `scan_manager.c::scan_open_heap_scan` from `total_pages / NUMBER_OF_SAMPLING_PAGES` (see [[components/scan-manager]]).

The sampling branch is gated on the `S_HEAP_SAMPLING_SCAN` scan type which is produced by the `/*+ SAMPLING_SCAN */` query hint; partitioned tables bypass sampling at scan-open time.

## Related

- Parent: [[components/storage|storage]]
- [[components/page-buffer]] — all page access
- [[components/btree]] — index keys derived from heap attributes
- [[components/overflow-file]] — storage for large records
- [[components/transaction]] — MVCC visibility, locks
- [[components/external-storage]] — LOB data via `es_delete_file`
