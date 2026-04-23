---
type: index
title: "CUBRID Components"
updated: 2026-04-23
tags:
  - index
  - cubrid
  - component
status: active
---

# Components

Reusable / functional components inside CUBRID modules. One page per subsystem or meaningful abstraction (e.g. query optimizer, lock manager, page buffer, log recovery, HA replication).

Navigation: [[index]] | [[modules/_index|Modules]] | [[Architecture Overview]]

---

## Storage Layer (`src/storage/`)

- [[components/storage|storage]] — overview: buffer pool, B-tree, heap, file/disk management, external sort, LOB
- [[components/page-buffer|page-buffer]] — buffer pool, LRU zones, fix/unfix, dirty tracking, DWB integration
- [[components/btree|btree]] — B+tree index: find, range scan, insert, MVCC delete, bulk load, unique stats
- [[components/heap-file|heap-file]] — row storage, MVCC paths, scan cache, class representation cache
- [[components/file-manager|file-manager]] — three-layer I/O stack: file_manager, disk_manager, file_io
- [[components/double-write-buffer|double-write-buffer]] — torn-write protection; DWB_SLOT; crash recovery ordering
- [[components/overflow-file|overflow-file]] — linked-page overflow for large heap records and long B-tree keys
- [[components/extendible-hash|extendible-hash]] — disk-resident extendible hash (internal use)
- [[components/external-sort|external-sort]] — external merge sort; `sort_listfile` entry point; parallel sort bridge
- [[components/external-storage|external-storage]] — LOB external storage API: POSIX, OWFS, LOCAL backends

---

## Query Execution Layer (`src/query/`)

- [[components/query|query]] — hub page: full XASL execution layer overview
- [[components/query-executor|query-executor]] — `qexec_execute_mainblock`, XASL dispatch, hash GROUP BY
- [[components/scan-manager|scan-manager]] — unified scan abstraction (15 scan types)
- [[components/cursor|cursor]] — client-side result cursor over QFILE_LIST_ID
- [[components/partition-pruning|partition-pruning]] — runtime partition elimination and DML routing
- [[components/dblink|dblink]] — remote CUBRID query (CCI) + 2-phase commit
- [[components/list-file|list-file]] — temp result spool, sort, set ops, result cache
- [[components/aggregate-analytic|aggregate-analytic]] — GROUP BY / window functions
- [[components/filter-pred-cache|filter-pred-cache]] — compiled filter predicate cache (filtered indexes)
- [[components/memoize|memoize]] — subquery result memoization

## Parallel Query (`src/query/parallel/`)

- [[components/parallel-query|parallel-query]] — parallel execution subsystem hub
- [[components/parallel-worker-manager|parallel-worker-manager]] — pool lifecycle and reservation
- [[components/parallel-task-queue|parallel-task-queue]] — MPMC queue and callable_task
- [[components/parallel-hash-join|parallel-hash-join]] — hash join parallelism
- [[components/parallel-heap-scan|parallel-heap-scan]] — heap scan parallelism
- [[components/parallel-query-execute|parallel-query-execute]] — subquery parallelism
- [[components/parallel-sort|parallel-sort]] — external sort parallelism

## Parser (`src/parser/`)

- [[components/parser|parser]] — SQL frontend hub: lexer → bison → PT_NODE → name resolution → semantic check → XASL generation
- [[components/parse-tree|parse-tree]] — PT_NODE tagged-union node; PARSER_CONTEXT arena; traversal API
- [[components/name-resolution|name-resolution]] — SCOPES stack; identifier → DB_OBJECT binding; class hierarchy flattening
- [[components/semantic-check|semantic-check]] — structural validation; union compatibility; expression type inference (type_checking.c)
- [[components/xasl-generation|xasl-generation]] — SYMBOL_INFO/TABLE_INFO scope stack; PT_NODE → XASL_NODE emission
- [[components/view-transform|view-transform]] — mq_translate view inlining; sargable term pushdown; updatability analysis
- [[components/parser-allocator|parser-allocator]] — parser_block_allocator; arena lifetime model; dealloc no-op
- [[components/show-meta|show-meta]] — SHOWSTMT_METADATA registry; DBA-only guard; per-type semantic check hooks
