---
type: flow
actors: [client, broker/CAS, db_server]
entry_point: "INSERT/UPDATE/DELETE/MERGE SQL statement"
exit_point: "modified rows + commit ack"
triggers: [user SQL]
related_modules: [parser, query, storage, transaction]
tags:
  - flow
  - cubrid
  - dml
  - execution
related:
  - "[[Query Processing Pipeline]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[components/parser|parser]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[components/xasl-stream|xasl-stream]]"
  - "[[components/communication|communication]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/btree|btree]]"
  - "[[components/mvcc|mvcc]]"
  - "[[components/lock-manager|lock-manager]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/db-value|db-value]]"
  - "[[components/vacuum|vacuum]]"
created: 2026-04-23
updated: 2026-04-23
---

# DML Execution Path — INSERT / UPDATE / DELETE / MERGE

## Summary

A CUBRID DML statement travels through three distinct environments: the client process compiles SQL all the way to an XASL plan (lex → parse → name resolution → type check → plan generation), the CAS broker serializes that plan and ships it to the database server via `NET_SERVER_QM_QUERY_EXECUTE`, and the server deserializes, dispatches through `qexec_execute_mainblock`, then mutates heap and index storage under MVCC, WAL, and lock discipline before returning an ack. UPDATE and DELETE additionally create dead row versions that the vacuum daemon reclaims asynchronously.

---

## End-to-End Sequence Diagram

```
 Client process (CS_MODE)          cub_broker/CAS          cub_server
 ─────────────────────────         ──────────────          ─────────────────────────────────────
 SQL text
   │
   ├─ csql_lexer.l      (tokenise)
   ├─ csql_grammar.y    (PT_NODE tree)
   ├─ name_resolution   (bind PT_NAME → DB_OBJECT)
   ├─ semantic_check    (validate structure)
   ├─ type_checking     (infer PT_TYPE_ENUM)
   ├─ view_transform    (inline views)
   └─ xasl_generation  (PT_NODE → XASL_NODE)
          │
          │  xts_map_xasl_to_stream()
          │  [flat byte stream]
          ▼
        NET_SERVER_QM_QUERY_EXECUTE ──────────────────►
                                                        stx_map_stream_to_xasl()
                                                        qexec_execute_query()
                                                          └─ qexec_execute_mainblock()
                                                               │
                                                         ┌─────┴──────────────────┐
                                                         │  dispatch on XASL.proc │
                                                         └─────┬──────────────────┘
                                                               │
                                              ┌────────────────┼────────────────────┐
                                           INSERT           UPDATE/DELETE          MERGE
                                              │                  │                   │
                                         lock_object       MVCC visibility     INSERT + DELETE
                                         heap_insert       lock_object         sub-flows
                                         btree_insert      heap_update/delete
                                         log_append        btree_mvcc_delete
                                              │            log_append
                                              └────────────────┴────────────────────┘
                                                               │
                                                          log_commit()
                                                          lock_unlock_all()
          ◄────────────────────────────── response (affected rows count)
```

---

## Stage 1: Client Compile

> [!key-insight] Parser is client-side only
> Every file in `src/parser/` is `#if defined(SERVER_MODE) #error`. The server never sees SQL text — only a serialized XASL byte stream arrives. See [[Build Modes (SERVER SA CS)]].

`pt_compile()` in `compile.c` orchestrates five sub-passes over the [[components/parse-tree|PT_NODE]] tree:

| Pass | File | What happens |
|------|------|-------------|
| `pt_resolve_names` | `name_resolution.c` | Bind `PT_NAME` nodes to `DB_OBJECT*`; set `spec_id` cross-refs |
| `pt_semantic_check` | `semantic_check.c` | Structural validation, union compatibility, view cycle detection |
| `pt_semantic_type` | `type_checking.c` | Infer `PT_TYPE_ENUM` on every node; resolve `PT_TYPE_MAYBE` host vars |
| `mq_translate` | `view_transform.c` | Inline views; rewrite method calls |
| `xasl_generate_statement` | `xasl_generation.c` | Emit `XASL_NODE` tree |

`xasl_generate_statement` dispatches on statement `node_type`:
- `PT_INSERT` → `pt_to_insert_xasl`
- `PT_UPDATE` → `pt_to_update_xasl`
- `PT_DELETE` → `pt_to_delete_xasl`
- `PT_MERGE`  → `pt_to_merge_xasl`

The optimizer (`query_planner.c`) is consulted to select access methods (heap scan vs. index scan). All expression nodes are converted to `REGU_VARIABLE` via `pt_to_regu_variable`; [[components/db-value|DB_VALUE]] literals are embedded directly in the XASL tree.

See [[components/parser|parser]] and [[components/xasl-generation|xasl-generation]] for full detail.

---

## Stage 2: Wire Transmission

The client packs the `XASL_NODE` tree to a flat byte buffer via `xts_map_xasl_to_stream()`. Pointers become integer offsets from the buffer start; all fields are 8-byte aligned. The stream header carries: creator OID, class OID list (for cache invalidation), `repr_id_list`, and `dbval_cnt`.

```
Client: xts_map_xasl_to_stream(xasl_node) → XASL_STREAM { buffer, size }
         │
         │  NET_SERVER_QM_QUERY_EXECUTE (sqmgr_execute_query handler)
         │  action_attribute: SET_DIAGNOSTICS_INFO | IN_TRANSACTION
         ▼
Server: stx_map_stream_to_xasl(thread_p, &xasl_tree, use_clone, buf, size, &unpack_info)
```

> [!key-insight] Visited-pointer dedup on unpack
> Shared sub-structures (e.g. a `REGU_VARIABLE` referenced multiple times) are allocated once on unpack. The `XASL_UNPACK_INFO` visited-pointer hash table prevents duplicate allocation. Pre-allocate the arena at `3 × stream_size` bytes (`UNPACK_SCALE` constant) — under-allocation causes mid-unpack realloc that invalidates all `packed_xasl` pointers.

See [[components/xasl-stream|xasl-stream]] and [[components/communication|communication]] for stream layout and dispatch table details.

---

## Stage 3: Server Dispatch

`qexec_execute_query()` looks up or creates `XASL_STATE`, checks the result cache, then calls `qexec_execute_mainblock()`. The proc type in `xasl->proc_type` selects the DML branch:

| `XASL.proc_type` | Branch |
|-----------------|--------|
| `INSERT_PROC` | INSERT path |
| `UPDATE_PROC` | UPDATE path |
| `DELETE_PROC` | DELETE path |
| `MERGE_PROC` | MERGE path (INSERT + DELETE sub-flows) |

All DML branches share the same lock → heap-mutate → index-maintain → WAL structure described below. See [[components/query-executor|query-executor]] for the full XASL node type table.

---

## Stage 4: Per-Statement Specifics

### 4.1 INSERT

**Lock acquisition**
1. `SCH_S_LOCK` on the target class (schema stability).
2. `IX_LOCK` (intent exclusive) on the class.
3. No per-row lock for new rows — the inserting transaction's MVCC ID serves as the visibility guard until commit.

**Heap mutation**
`heap_insert_logical(thread_p, context, home_hint_p)` using a `HEAP_OPERATION_CONTEXT` created by `heap_create_insert_context`. The new record gets `MVCC_REC_HEADER.mvcc_ins_id` stamped with the current transaction's MVCC ID. For records exceeding a page, `overflow_insert` writes overflow pages and leaves a `REC_BIGONE` slot in the home page.

**Index maintenance**
For every index on the table, `btree_insert(thread_p, btid, key, cls_oid, oid, BTREE_OP_INSERT_NEW_OBJECT, ...)` adds the `(key, OID)` pair with the insert MVCCID. Unique constraints are checked at statement end via `multi_index_unique_stats` accumulated in `HEAP_SCANCACHE`.

**WAL**
`log_append_undoredo_data(thread_p, RVHF_MVCC_INSERT, addr, undo_len, redo_len, undo, redo)` writes a `LOG_MVCC_UNDOREDO_DATA` record before any page is dirtied. [[components/page-buffer|page-buffer]] enforces ordering: `pgbuf_flush_with_wal` ensures the log record reaches disk before the page does.

> [!key-insight] The parser never sees the row being inserted
> At compile time, XASL generation only knows the schema; the actual OID and page location are unknown until `heap_insert_logical` runs on the server and assigns an OID via `heap_assign_address`.

### 4.2 UPDATE

UPDATE is MVCC-delete-old + MVCC-insert-new. The two sub-operations share a single transaction but may touch different pages.

**Read path (source rows)**
The executor opens a scan via [[components/scan-manager|scan-manager]] (`scan_open_heap_scan` or `scan_open_index_scan` depending on the access method chosen by the optimizer). For each candidate row, [[components/mvcc|mvcc]] visibility is checked:

```
mvcc_satisfies_delete(thread_p, rec_header)
  → DELETE_RECORD_CAN_DELETE   ← proceed
  → DELETE_RECORD_DELETED      ← skip (already gone)
  → DELETE_RECORD_DELETE_IN_PROGRESS ← block and retry
```

**Lock acquisition**
- `SCH_S_LOCK` on class.
- `IX_LOCK` on class.
- `X_LOCK` on each candidate row OID (`lock_object(thread_p, oid, class_oid, X_LOCK, LK_UNCOND_LOCK)`).

> [!key-insight] MVCC UPDATE = insert new version + delete old version
> `heap_update_logical` does not overwrite in place. It stamps `mvcc_del_id` on the old record (making it a dead version for concurrent readers) and calls `heap_insert_logical` for the new version. Both versions coexist until vacuum reclaims the dead one. The `prev_version_lsa` field in the new record's MVCC header points back to the old version's log LSA for version-chain traversal.

**Heap mutation**
`heap_update_logical(thread_p, context)`:
1. Fix old page with `PGBUF_LATCH_WRITE`.
2. Set `MVCC_REC_HEADER.mvcc_del_id` = current MVCCID on old record → `LOG_MVCC_UNDOREDO_DATA`.
3. Insert new record (may land on same or different page).

**Index maintenance**
Old key: `btree_mvcc_delete(thread_p, btid, old_key, class_oid, oid, BTREE_OP_INSERT_MVCC_DELID, ...)` — stamps delete MVCCID in the leaf entry without physical removal.
New key (if key value changed): `btree_insert(thread_p, btid, new_key, ..., BTREE_OP_INSERT_NEW_OBJECT, ...)`.

**WAL**
Two `log_append_undoredo_data` calls (one per heap op). See [[components/log-manager|log-manager]] for `LOG_MVCC_UNDOREDO_DATA` record layout including `LOG_VACUUM_INFO.prev_mvcc_op_log_lsa` — the chain pointer the vacuum daemon follows.

### 4.3 DELETE

**Read path** — same MVCC visibility check as UPDATE (`mvcc_satisfies_delete`).

**Lock acquisition** — `SCH_S_LOCK` → `IX_LOCK` on class → `X_LOCK` per row.

**Heap mutation**
`heap_delete_logical(thread_p, context)` stamps `mvcc_del_id` in the record header. The slot is *not* freed immediately — it remains as a dead version visible to earlier snapshots. LOB attributes are unlinked via `heap_attrinfo_delete_lob` → `es_delete_file`.

**Index maintenance**
`btree_mvcc_delete` with `BTREE_OP_INSERT_MVCC_DELID` for every index on the table.

**WAL** — `log_append_undoredo_data` with `RVHF_MVCC_DELETE` recovery index.

> [!key-insight] Old versions live in the WAL, not the heap
> Unlike PostgreSQL, CUBRID stores old row versions as undo images in WAL log records. The `prev_version_lsa` chain in each MVCC header lets readers walk backward through log pages to find the version visible to their snapshot. Vacuum later clears `prev_version_lsa` once no active snapshot can reach that version.

### 4.4 MERGE

MERGE (`ON DUPLICATE KEY`-style upsert) compiles to a `MERGE_PROC` XASL node generated by `pt_to_merge_xasl`. At execution time, `qexec_execute_mainblock` runs the WHEN MATCHED branch (UPDATE sub-flow, §4.2) and the WHEN NOT MATCHED branch (INSERT sub-flow, §4.1) based on join results from the source scan. Both branches acquire locks independently; the lock order (class intent before row exclusive) is the same.

---

## Stage 5: Commit

```
log_commit(thread_p, tran_index, retain_lock=false)
  ├── log_do_postpone()          ← execute deferred constraint checks
  ├── log_complete()
  │     └── write LOG_COMMIT record to log buffer
  ├── log_flush_if_needed()      ← flush log up to commit LSA
  └── lock_unlock_all(thread_p)  ← release all transaction locks
```

Only after `LOG_COMMIT` reaches disk (WAL) are locks released. The response packet (affected row count) is then sent back to the client via the `NET_SERVER_QM_QUERY_EXECUTE` reply path.

---

## Stage 6: Vacuum Aftermath

UPDATE and DELETE leave dead row versions in heap pages. The [[components/vacuum|vacuum]] daemon reclaims them asynchronously:

```
vacuum master (TT_VACUUM_MASTER)
  │  follows LOG_VACUUM_INFO.prev_mvcc_op_log_lsa chain
  │  backward through MVCC log records
  ▼
vacuum worker (TT_VACUUM_WORKER) × up to 50
  ├─ heap page: mvcc_satisfies_vacuum() → VACUUM_RECORD_REMOVE
  │    └─ physically remove slot (mark free) + log RVHF_* record
  ├─ heap page: → VACUUM_RECORD_DELETE_INSID_PREV_VER
  │    └─ clear mvcc_ins_id + prev_version_lsa; set MVCCID_ALL_VISIBLE
  └─ btree: btree_vacuum_object() / btree_vacuum_insert_mvccid()
       └─ remove stale MVCC markers from leaf entries
```

Vacuum checks `oldest_mvccid` (minimum across all active snapshots) before reclaiming. Each vacuum worker holds a private LRU partition in [[components/page-buffer|page-buffer]] so it cannot evict hot data pages.

---

## Failure Modes

### Deadlock
The deadlock daemon runs inside [[components/lock-manager|lock-manager]]. When a cycle is detected, the **youngest** transaction (highest `tranid`) is selected as victim — it receives `LK_NOTGRANTED_DUE_ABORTED`. The victimized transaction proceeds to abort (see below).

### Abort / Rollback
`log_abort(thread_p, tran_index)` walks the transaction's log chain (`prev_tranlsa`) in reverse order:
1. For each `LOG_UNDOREDO_DATA` / `LOG_MVCC_UNDOREDO_DATA`: apply undo (call `rcvindex` undo handler, e.g. `heap_rv_undo_insert`, `btree_rv_keyval_undo_insert`).
2. Write `LOG_COMPENSATE` (CLR) record for each undone action.
3. Write `LOG_ABORT` at the end.

B-tree operations protected by `LOG_SYSOP_ATOMIC_START` (type 50) are rolled back immediately if encountered incomplete during redo — preventing partial page splits from persisting on disk.

### Torn Writes
Before any dirty page reaches its volume location, it passes through the [[components/page-buffer|double-write buffer]] (`dwb_add_page`). The DWB block is `fsync`ed first; only then is the page written to the actual data file. This ensures an incomplete write is detectable and recoverable via DWB replay on restart.

---

## Cross-References

| Component | Role in DML |
|-----------|------------|
| [[components/parser|parser]] | SQL → PT_NODE (client only) |
| [[components/parse-tree|parse-tree]] | PT_NODE struct; `PT_INSERT/UPDATE/DELETE/MERGE` info members |
| [[components/xasl-generation|xasl-generation]] | PT_NODE → XASL_NODE; `pt_to_insert_xasl` etc. |
| [[components/xasl-stream|xasl-stream]] | XASL serialization; `xts_map_xasl_to_stream` / `stx_map_stream_to_xasl` |
| [[components/communication|communication]] | `NET_SERVER_QM_QUERY_EXECUTE` dispatch; `sqmgr_execute_query` handler |
| [[components/query-executor|query-executor]] | `qexec_execute_mainblock`; INSERT/UPDATE/DELETE/MERGE_PROC branches |
| [[components/scan-manager|scan-manager]] | Source row iteration for UPDATE/DELETE/MERGE; `scan_op_type` controls MVCC lock mode |
| [[components/heap-file|heap-file]] | `heap_insert/update/delete_logical`; MVCC record header; overflow records |
| [[components/btree|btree]] | `btree_insert` / `btree_mvcc_delete`; `btree_op_purpose` enum |
| [[components/mvcc|mvcc]] | `mvcc_satisfies_delete`; snapshot visibility; version chain via `prev_version_lsa` |
| [[components/lock-manager|lock-manager]] | `lock_object`; SCH-S / IX / X lock hierarchy; deadlock detection |
| [[components/log-manager|log-manager]] | `log_append_undoredo_data`; `LOG_MVCC_UNDOREDO_DATA`; `log_commit` / `log_abort` |
| [[components/page-buffer|page-buffer]] | `pgbuf_fix/unfix/set_dirty`; WAL-safe flush; DWB integration |
| [[components/db-value|db-value]] | `DB_VALUE` carries column values through parse → XASL → executor → storage |
| [[components/vacuum|vacuum]] | Asynchronous reclaim of dead heap rows and stale index keys |

---

[[flows/_index|Flows]] | [[Query Processing Pipeline]] | [[Data Flow]]
