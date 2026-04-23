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

## Transaction Layer (`src/transaction/`)

- [[components/transaction|transaction]] — hub: MVCC, WAL, locking, deadlock detection, recovery, boot, vacuum
- [[components/mvcc|mvcc]] — `MVCC_REC_HEADER`, snapshot, `mvcc_satisfies_snapshot` visibility predicate
- [[components/lock-manager|lock-manager]] — `LK_RES`/`LK_ENTRY`, 8 lock modes, hierarchical acquire/release, escalation
- [[components/deadlock-detection|deadlock-detection]] — wait-for graph, DFS cycle finder, youngest-victim policy
- [[components/log-manager|log-manager]] — WAL append path, `LOG_RECORD_HEADER`, 52 record types, checkpoint, commit/abort
- [[components/recovery|recovery]] — ARIES crash recovery: analysis → redo → undo; CLR; 2PC; atomic sysops
- [[components/vacuum|vacuum]] — MVCC GC daemon (in `src/query/`); master + up-to-50 workers; heap/btree cleanup
- [[components/server-boot|server-boot]] — subsystem init order, `BOOT_DB_PARM`, crash-recovery entry point

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

## XASL (`src/xasl/` + `src/query/xasl.h`)

- [[components/xasl|xasl]] — XASL hub: eXecutable Algebraic Statement Language; XASL_NODE plan tree; client→server serialisation; PROC_TYPE enum
- [[components/xasl-stream|xasl-stream]] — serialisation protocol: offset-based pointer encoding, stx_build/stx_restore, XASL_UNPACK_INFO visited-pointer table
- [[components/regu-variable|regu-variable]] — REGU_VARIABLE expression atom: 17-way discriminated union (attr, arith, func, subquery, host-var, …)
- [[components/xasl-predicate|xasl-predicate]] — PRED_EXPR boolean tree: AND/OR/NOT combinators; COMP/ALSM/LIKE/RLIKE eval-term leaves
- [[components/xasl-aggregate|xasl-aggregate]] — AGGREGATE_TYPE: aggregate function nodes with serialised fields + server-only accumulator
- [[components/xasl-analytic|xasl-analytic]] — ANALYTIC_TYPE / ANALYTIC_EVAL_TYPE: window function nodes grouped by compatible sort specs

## Parser (`src/parser/`)

- [[components/parser|parser]] — SQL frontend hub: lexer → bison → PT_NODE → name resolution → semantic check → XASL generation
- [[components/parse-tree|parse-tree]] — PT_NODE tagged-union node; PARSER_CONTEXT arena; traversal API
- [[components/name-resolution|name-resolution]] — SCOPES stack; identifier → DB_OBJECT binding; class hierarchy flattening
- [[components/semantic-check|semantic-check]] — structural validation; union compatibility; expression type inference (type_checking.c)
- [[components/xasl-generation|xasl-generation]] — SYMBOL_INFO/TABLE_INFO scope stack; PT_NODE → XASL_NODE emission
- [[components/view-transform|view-transform]] — mq_translate view inlining; sargable term pushdown; updatability analysis
- [[components/parser-allocator|parser-allocator]] — parser_block_allocator; arena lifetime model; dealloc no-op
- [[components/show-meta|show-meta]] — SHOWSTMT_METADATA registry; DBA-only guard; per-type semantic check hooks

---

## Threading Layer (`src/thread/`)

- [[components/thread|thread]] — hub: `cubthread` namespace, manager, worker pools, daemons, THREAD_ENTRY
- [[components/thread-manager|thread-manager]] — `cubthread::manager` singleton, pool/daemon registry, entry pool, `get_manager()`
- [[components/worker-pool|worker-pool]] — `worker_pool_type`, `execute`, `execute_on_core`, core partitioning, stats variant
- [[components/entry-task|entry-task]] — `entry_task` abstract base, retire pattern, `entry_manager`, `callable_task`
- [[components/thread-daemon|thread-daemon]] — daemon lifecycle, `looper` strategies (INF/FIXED/INCREASING/CUSTOM), known daemons

---

## Compat Layer (`src/compat/`)

- [[components/compat|compat]] — hub: public client API surface (`db_*` namespace) and `DB_VALUE` universal value container
- [[components/db-value|db-value]] — `DB_VALUE` tagged union: `DB_TYPE` enum (41 types), `DB_DATA` union, `need_clear` ownership, `db_make_*` / `db_get_*` patterns
- [[components/client-api|client-api]] — `db_*` families: connection, transaction, schema DDL, object CRUD, query compile/execute/fetch, LOB, sets
- [[components/dbi-compat|dbi-compat]] — `dbi_compat.h` umbrella header, `SQLX_CMD_*` alias layer, error-code mirror (place 2 of the 6-place rule)

---

## Base Utilities (`src/base/`)

- [[components/base|base]] — hub: error handling, memory, lock-free, porting, i18n, perf monitoring, serialization, system config
- [[components/error-manager|error-manager]] — `er_set`, severity levels, error stack, ASSERT_ERROR macros; `error_code.h` (~1700 codes)
- [[components/memory-alloc|memory-alloc]] — `db_private_alloc`, `free_and_init`, `memory_wrapper.hpp` placement rule, area/slab allocator
- [[components/lockfree|lockfree]] — `lockfree::hashmap<Key,T>` (modern) + `LF_HASH_TABLE` (legacy); epoch-based reclamation
- [[components/system-parameter|system-parameter]] — ~400 `PRM_ID_*` params; `prm_get_*_value()` API; reads `cubrid.conf`
- [[components/porting|porting]] — POSIX↔Win32 shims; `EXPORT_IMPORT`; dynamic library loading; `ONE_K`/`ONE_M` constants

---

## Message Catalog (`msg/`)

- [[components/message-catalog|message-catalog]] — POSIX `catgets`-style catalog format, `gencat` build pipeline, `msgcat_*` C loader; three catalogs (`cubrid`, `csql`, `utils`); four locales (`en_US.utf8`, `en_US`, `ko_KR.utf8`, `ko_KR.euckr`)

---

## Bundled 3rd-party (`src/heaplayers/`)

- [[components/heaplayers|heaplayers]] — Heap Layers (Emery Berger); `lea_heap.c` ~181 KB Doug Lea malloc; do-not-modify; excluded from cppcheck; SERVER_MODE LEA heap backend for `db_private_alloc`

---

## Broker Layer (`src/broker/`)

- [[components/broker-impl|broker-impl]] — hub: connection broker (multi-process router), CAS lifecycle, 3-tier topology, connection pooling
- [[components/cas|cas]] — CAS worker process: request loop, `server_fn_table[]` dispatch, db connection via CSS
- [[components/broker-shm|broker-shm]] — shared memory IPC: `T_SHM_BROKER`, `T_SHM_APPL_SERVER`, `T_APPL_SERVER_INFO`, semaphore protocol
- [[components/shard-broker|shard-broker]] — optional shard proxy: range/hash routing, `shard_*` files, `T_SHM_PROXY`

---

## Communication Layer (`src/communication/`)

- [[components/communication|communication]] — hub: NET_SERVER_REQUEST_LIST dispatch table, request handler registration, method/xs callback glue, per-request histogram
- [[components/packer|packer]] — `cubpacking::packer` / `unpacker`: type-safe binary serialization; variadic `pack_all` / `set_buffer_and_pack_all`; `packable_object` interface
- [[components/request-response|request-response]] — `net_request` struct, `net_req_act` bitmask flags, `net_server_func` handler contract, dispatch flow from CSS packet to function call

---

## Connection Layer (`src/connection/`)

- [[components/connection|connection]] — hub: CSS protocol, cub_master coordination, TCP + Unix sockets, HA heartbeat
- [[components/cub-master|cub-master]] — master process: dual-socket listen, FD passing, HA process registry, management commands
- [[components/network-protocol|network-protocol]] — `NET_HEADER` packet format, packet types, request ID encoding, css_error_code
- [[components/heartbeat|heartbeat]] — HA heartbeat: node states, 500 ms interval, 5-gap failover, HBP packet format, log applier tracking
- [[components/tcp-layer|tcp-layer]] — socket primitives: `css_tcp_client_open`, `css_tcp_master_open`, SCM_RIGHTS fd passing, `css_peer_alive`

---

## Object / Schema / Auth Layer (`src/object/`)

- [[components/object|object]] — hub: schema, auth, catalog, class representation, LOB locator, workspace, triggers
- [[components/schema-manager|schema-manager]] — class/table DDL lifecycle; SM_TEMPLATE edit-commit pattern; constraint management
- [[components/system-catalog|system-catalog]] — `_db_class` and friends; info-schema virtual views; CI-enforced 9-rule SQL formatting
- [[components/authenticate|authenticate]] — users, groups, privilege caching; `authenticate_context`; execution-rights stack for SPs
- [[components/lob-locator|lob-locator]] — LOB locator state machine (TRANSIENT/PERMANENT); CS/SA mode dispatch

---

## Stored Procedure Bridge (`src/sp/`)

- [[components/sp|sp]] — hub: C++ ↔ Java PL engine bridge; JVM lifecycle, connection pool, catalog DDL, error propagation
- [[components/sp-jni-bridge|sp-jni-bridge]] — invocation mechanics, DB_VALUE marshalling, unsupported types, interrupt handling
- [[components/sp-method-dispatch|sp-method-dispatch]] — XASL METHOD_CALL_NODE → cubpl::executor dispatch; recursion limit; OUT arg write-back
- [[components/sp-protocol|sp-protocol]] — UDS/TCP transport, SP_CODE opcodes, METHOD_CALLBACK bidirectional loop, epoch-based reconnect

---

## Method Invocation Layer (`src/method/`)

- [[components/method|method]] — hub: scan-time C method + SP invocation; S_METHOD_SCAN; client-side callback handler
- [[components/method-invoke-group|method-invoke-group]] — `cubmethod::method_invoke_group`: shared dispatch struct (used by both src/method/ and src/sp/)
- [[components/method-scan|method-scan]] — `cubscan::method::scanner`: S_METHOD_SCAN backend wired into scan-manager

---

## Bulk Loader (`src/loaddb/`)

- [[components/loaddb|loaddb]] — hub: `loaddb` utility bulk loader; own bison/flex grammar; parallel batch processing; direct heap insert bypassing SQL execution
- [[components/loaddb-grammar|loaddb-grammar]] — `load_grammar.yy` LALR(1) C++ bison grammar + `load_lexer.l` flex scanner; event-driven (no parse tree)
- [[components/loaddb-executor|loaddb-executor]] — `server_class_installer`, `server_object_loader`; string→DB_VALUE dispatch; `locator_multi_insert_force` bulk insert path
- [[components/loaddb-driver|loaddb-driver]] — `driver` (scanner+parser orchestration), `session` (lifecycle + ordered batch-commit), `worker_entry_manager` (driver pool per thread)

---

## Session Layer (`src/session/`)

- [[components/session|session]] — hub: per-connection state container; session variables, prepared statements, holdable cursors, session params, PL session, private LRU partition
- [[components/session-state|session-state]] — `SESSION_STATE` struct & lifecycle: create/check/destroy, ref_count protocol, 60-second timeout reaper, lock-free hashmap
- [[components/session-variables|session-variables]] — `@var` user variable bindings (linked list, max 20) and session-level system parameter overrides (`SESSION_PARAM` array, O(1) `prm_Def_session_idx` lookup)

---

## Performance Monitor (`src/monitor/`)

- [[components/monitor|monitor]] — hub: runtime perf statistics; `statistic_value` wire type; global named registry; per-transaction sheet tracking; VACUUM ovfp threshold (server-only)
- [[components/perfmon|perfmon]] — core API: primitive/atomic templates (accumulator, gauge, max, min), composite stats (`counter_timer_statistic`), autotimer RAII, name builders
- [[components/stats-collection|stats-collection]] — aggregation model: always-on global counters + optional per-transaction sheet isolation; snapshot-delta pattern; overhead characteristics

---

## Debug Utilities (`src/debugging/`)

- [[components/debugging|debugging]] — `type_helper.hpp`: compile-time type name stringification; `strict_warnings` (referenced, not yet in tree); zero runtime cost, debug-build-only

---

## Public C API Extensions (`src/api/`)

- [[components/api|api]] — hub: public C API extensions beyond `db_*`; CS_MODE-only; currently hosts the CDC interface
- [[components/cubrid-log-cdc|cubrid-log-cdc]] — `cubrid_log.h/c`: CDC client API; four-phase state machine; DDL/DML/DCL/TIMER log item stream; LSA-based positioning; `supplemental_log` prerequisite

---

## Executables (`src/executables/`)

- [[components/executables|executables]] — hub: all CUBRID binary entry points, binary inventory, build-mode link split
- [[components/cub-server-main|cub-server-main]] — `server.c` entry point: signal handlers, `net_server_start()` handoff
- [[components/csql-shell|csql-shell]] — CSQL REPL: DSO-based SA/CS mode switch, session commands, readline, result display
- [[components/cub-master-main|cub-master-main]] — `master.c` main loop: `select()` event loop, connection dispatch, auto-restart
- [[components/utility-binaries|utility-binaries]] — admin utilities: createdb, unloaddb, compactdb, backupdb, killtran, cubrid service front-end

---

## Windows Tools (`src/win_tools/`) — Windows only

- [[components/win-tools|win-tools]] — hub: NT service host (`CUBRIDService`), CLI control client (`ctrlservice`), MFC system-tray app (`CUBRID_Service_Tray`)
