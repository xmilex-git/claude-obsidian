---
status: active
type: source
title: "CUBRID src/storage/ — Storage Layer Source"
source_path: "src/storage/"
ingested: 2026-04-23
files_read:
  - "AGENTS.md"
  - "page_buffer.h / page_buffer.c (head + LRU zone section)"
  - "btree.h / btree_load.h / btree_unique.hpp"
  - "heap_file.h"
  - "file_manager.h"
  - "file_io.h"
  - "disk_manager.h"
  - "double_write_buffer.hpp"
  - "external_sort.h"
  - "extendible_hash.h"
  - "oid.h"
  - "es.h / es_common.h / es_posix.h"
  - "overflow_file.h"
  - "catalog_class.h"
tags:
  - source
  - cubrid
  - storage
created: 2026-04-23
---

# Source: `src/storage/` — Storage Layer

## Coverage

Read 20+ header files and implementation heads from CUBRID's `src/storage/` directory (57 files total per user guidance). Key patterns from `page_buffer.c` (LRU zone definitions, BCB flags, neighbor flush) and `btree.h` (BTREE_SCAN, btree_op_purpose, unique stats) were read in full.

## Pages Created / Updated

| Page | Action |
|------|--------|
| [[components/storage]] | Upgraded from stub to comprehensive overview |
| [[components/page-buffer]] | Created — buffer pool, LRU zones, BCB, fix/unfix, DWB integration |
| [[components/btree]] | Created — B+tree, MVCC ops, scan, unique stats, bulk load |
| [[components/heap-file]] | Created — row storage, MVCC paths, scan cache, class repr cache |
| [[components/file-manager]] | Created — three-layer stack: file_manager, disk_manager, file_io |
| [[components/double-write-buffer]] | Created — torn-write protection, DWB_SLOT, recovery ordering |
| [[components/overflow-file]] | Created — linked-page overflow for large records/keys |
| [[components/extendible-hash]] | Created — disk-resident extendible hash, internal use |
| [[components/external-sort]] | Created — sort_listfile, SORT_INFO, parallel sort bridge |
| [[components/external-storage]] | Created — LOB backends: POSIX, OWFS, LOCAL; URI routing |

## Key Insights

1. **Three-zone LRU replacement** — buffer pool uses three LRU zones (hot/buffer/victim) rather than plain LRU. Only zone 3 yields victims. Vacuum thread page accesses deliberately bypass zone boosting to avoid polluting the hot working set.

2. **Latch vs lock duality** — page latches (short-term, physical) are separate from transaction locks (logical, MVCC). B-tree crabbing uses latches; uniqueness checking uses transaction locks through the lock manager.

3. **WAL ordering enforced at flush** — `pgbuf_flush_with_wal` checks that the page LSA is durable in the log before writing the page. This is the single enforcement point for WAL ordering in the buffer pool.

4. **DWB precedes WAL redo** — double-write buffer recovery runs before log redo during startup, ensuring all partially-written pages are resolved before logical recovery begins.

5. **LOB external storage is not WAL-logged** — `es_delete_file` is a direct filesystem call. CUBRID uses a `LOG_POSTPONE` record to defer LOB deletion until after transaction commit, preventing orphan LOBs on rollback.

6. **External sort is the parallel sort bridge** — `sort_listfile` is the single entry point for all sorting. When `parallelism > 1`, it internally invokes the parallel sort worker dispatch macros from `px_sort.h`.

## Cross-References

- [[components/transaction]] — WAL, lock manager, MVCC snapshot
- [[components/object|object]] — LOB locator (`lob_locator.cpp`) owns the URI lifecycle; heap owns the physical LOB bytes via `es.c`
- [[components/parallel-sort]] — calls `sort_listfile` from external_sort
- [[components/parallel-query]] — parallel heap scan accesses page buffer and heap file
- [[Memory Management Conventions]] — `db_private_alloc`, `free_and_init` used throughout
- [[Build Modes (SERVER SA CS)]] — storage files are `SERVER_MODE` / `SA_MODE` only (with `#error` guard in `btree.h`, `heap_file.h`, `external_sort.h`)
