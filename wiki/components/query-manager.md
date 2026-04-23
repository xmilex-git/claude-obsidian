---
type: component
title: "query-manager — Query Lifecycle and Per-Query Context"
parent_module: "[[modules/src|src]]"
path: src/query/query_manager.{c,h}
status: developing
key_files:
  - src/query/query_manager.c
  - src/query/query_manager.h
public_api:
  - qmgr_initialize
  - qmgr_finalize
  - qmgr_allocate_tran_entries
  - xqmgr_execute_query
  - xqmgr_prepare_query
  - xqmgr_end_query
  - qmgr_get_query_entry
  - qmgr_is_query_interrupted
  - qmgr_clear_trans_wakeup
  - qmgr_create_new_temp_file
  - qmgr_get_new_page / qmgr_get_old_page / qmgr_free_old_page
  - qmgr_get_sql_id
  - qmgr_get_rand_buf
tags:
  - cubrid
  - query
  - lifecycle
  - server-side
  - temp-file
---

# query-manager — Query Lifecycle and Per-Query Context

> [!key-insight]
> The query manager is the **server-side registry** for in-flight queries. It owns the per-transaction query entry table (`QMGR_QUERY_TABLE`), the temporary file infrastructure (memory-buffered + disk-backed), and the cancellation/interrupt signal path. The actual query execution (`qexec_execute_query`) is **called from inside** `qmgr_process_query` — query-manager wraps executor, not the other way around.

## Purpose

`query_manager.c` manages the full server-side lifecycle of a query:

1. **Query ID generation**: monotonically incrementing per transaction, stored in `QMGR_TRAN_ENTRY.query_id_generator`.
2. **Query entry registry**: `QMGR_QUERY_ENTRY` linked list per transaction; allows locating active queries for cancellation, temp-file cleanup, and result retrieval.
3. **Temporary file management**: `QMGR_TEMP_FILE` with in-memory buffer (`membuf`) + optional disk file (`VFID`). Two buffer types: `TEMP_FILE_MEMBUF_NORMAL` and `TEMP_FILE_MEMBUF_KEY_BUFFER`.
4. **XASL cache + result cache integration**: `xqmgr_execute_query` checks the XASL cache for a compiled plan, checks the list-file result cache, and falls through to `qmgr_process_query` on a miss.
5. **Interrupt handling**: `qmgr_is_query_interrupted` checks if the current query has been marked for cancellation (via `qmgr_set_query_error`).
6. **DBLink connection management**: per-transaction `DBLINK_CONN_ENTRY` tracking.

---

## Key Structures

```c
QMGR_QUERY_TABLE (global singleton)
  └─ QMGR_TRAN_ENTRY tran_entries_p[num_trans]
       ├─ QMGR_TRAN_STATUS trans_stat  (NULL/RUNNING/DELAYED/WAITING/TERMINATED/…)
       ├─ int query_id_generator        (monotonic, per-transaction)
       ├─ QMGR_QUERY_ENTRY *query_entry_list_p
       ├─ QMGR_QUERY_ENTRY *free_query_entry_list_p
       ├─ DBLINK_CONN_ENTRY *dblink_entry
       ├─ OID_BLOCK_LIST *modified_classes_p
       └─ pthread_mutex_t mutex

QMGR_QUERY_ENTRY
  ├─ QUERY_ID query_id
  ├─ XASL_ID xasl_id
  ├─ xasl_cache_ent *xasl_ent
  ├─ QFILE_LIST_ID *list_id        (result)
  ├─ QFILE_LIST_CACHE_ENTRY *list_ent
  ├─ QMGR_TEMP_FILE *temp_vfid     (head of per-query temp file chain)
  ├─ int num_tmp, total_count
  ├─ char *er_msg, int errid       (error captured mid-execution)
  ├─ QMGR_QUERY_STATUS query_status (IN_PROGRESS / COMPLETED / CLOSED)
  ├─ QUERY_FLAG query_flag
  ├─ bool is_holdable
  └─ bool includes_tde_class

QMGR_TEMP_FILE
  ├─ FILE_TYPE temp_file_type
  ├─ VFID temp_vfid
  ├─ PAGE_PTR *membuf              (array of in-memory pages)
  ├─ int membuf_last, membuf_npages
  ├─ QMGR_TEMP_FILE_MEMBUF_TYPE membuf_type
  ├─ bool preserved               (not freed on query end if holdable)
  └─ bool tde_encrypted
```

---

## Public Entry Points

| Signature | Role |
|-----------|------|
| `int qmgr_initialize(THREAD_ENTRY*)` | Init global query table; allocate `MAX_NTRANS` tran entries |
| `void qmgr_finalize(THREAD_ENTRY*)` | Free all temp files and query entries |
| `int qmgr_allocate_tran_entries(THREAD_ENTRY*, int trans_cnt)` | (Re)allocate tran entry array; called by boot on connection pool resize |
| `QFILE_LIST_ID* xqmgr_execute_query(…XASL_ID*, QUERY_ID*, …)` | Main server-side entry: XASL cache lookup → list cache → execute |
| `int xqmgr_prepare_query(…compile_context*, …xasl_stream*)` | Register XASL byte stream in server XASL cache; return XASL_ID |
| `int xqmgr_end_query(THREAD_ENTRY*, QUERY_ID)` | Mark query completed; free temp files |
| `QMGR_QUERY_ENTRY* qmgr_get_query_entry(THREAD_ENTRY*, QUERY_ID, int trans_ind)` | Lookup query entry by ID |
| `bool qmgr_is_query_interrupted(THREAD_ENTRY*, QUERY_ID)` | SERVER_MODE only: check cancel flag |
| `void qmgr_set_query_error(THREAD_ENTRY*, QUERY_ID)` | Record error code into query entry |
| `void qmgr_clear_trans_wakeup(THREAD_ENTRY*, int tran_index, bool tran_died, bool is_abort)` | Transaction teardown: free all query entries + temp files for this tran |
| `QMGR_TEMP_FILE* qmgr_create_new_temp_file(THREAD_ENTRY*, QUERY_ID, membuf_type)` | Allocate new temp file (memory or disk) for a query |
| `PAGE_PTR qmgr_get_new_page(THREAD_ENTRY*, VPID*, QMGR_TEMP_FILE*)` | Get next writable page from temp file |
| `PAGE_PTR qmgr_get_old_page(THREAD_ENTRY*, VPID*, QMGR_TEMP_FILE*)` | Get existing page for reading |
| `void qmgr_free_old_page(THREAD_ENTRY*, PAGE_PTR, QMGR_TEMP_FILE*)` | Release page (nop for membuf, pgbuf_unfix for disk) |
| `void qmgr_add_modified_class(THREAD_ENTRY*, const OID*)` | Record class OID modified by DML (used to invalidate XASL cache) |
| `int qmgr_get_sql_id(THREAD_ENTRY*, char** buf, char* query, size_t len)` | Compute 13-char SQL hash ID (for query history log) |
| `QUERY_ID qmgr_get_current_query_id(THREAD_ENTRY*)` | Return query ID executing on current thread |

---

## Execution Path

```
Client sends NET_SERVER_QM_QUERY_EXECUTE (XASL_ID + host vars)
    xqmgr_execute_query(thread_p, xasl_id, &query_id, …)
        XASL cache lookup → xasl_cache_fix(xasl_id) → XASL_CLONE
        if result cache hit: return cached QFILE_LIST_ID
        qmgr_add_query_entry(…)          ← register new QMGR_QUERY_ENTRY
        qmgr_set_query_exec_info_to_tdes ← bind values → tdes history
        qmgr_process_query(thread_p, xasl_tree, …)
            stx_map_stream_to_xasl(…)    ← deserialize if no clone
            qexec_execute_query(…)        ← actual executor
            qfile_clone_list_id(result)   ← caller-owned copy
        qmgr_mark_query_as_completed(query_p)
        return list_id to client
```

---

## Temporary File Architecture

Two-tier storage per query:

```
QMGR_TEMP_FILE
  tier-1: membuf (PAGE_PTR array, malloc)
           TEMP_FILE_MEMBUF_NORMAL    → general sort/hash pages
           TEMP_FILE_MEMBUF_KEY_BUFFER → B-tree key buffer pages
  tier-2: VFID on disk (FILE_TEMP)
           created on-demand when membuf exhausted
           TDE-encrypted if includes_tde_class
```

`qmgr_get_page_type()` discriminates membuf vs disk page by pointer arithmetic on the membuf array bounds. Free list of pre-allocated `QMGR_TEMP_FILE` structs is maintained per type (`qmgr_Query_table.temp_file_list[type]`).

---

## Cancellation / Interrupt Hook

```c
// SERVER_MODE only
bool qmgr_is_query_interrupted(thread_p, query_id)
    // checks query_p->errid < 0 and thread interrupt flags
    // called periodically inside qexec inner loops
```

`qmgr_set_query_error(thread_p, query_id)` records the current `er_errid()` into `query_p->errid`. The executor polls `qmgr_is_query_interrupted` and returns `ER_INTERRUPTED` on detection.

> [!warning]
> `qmgr_is_query_interrupted` is a SERVER_MODE-only function (guarded by `#if defined(SERVER_MODE)`). SA_MODE uses a simpler interrupt flag on the thread entry. Do not call it in SA_MODE code paths.

---

## Query ID Generation

```c
// Per transaction, inside tran_entry_p->mutex:
query_id = ++tran_entry_p->query_id_generator;
// Uniqueness: (tran_index, query_id_generator) pair
// QUERY_ID is typedef'd as a 64-bit integer in storage_common.h
```

The `query_id_generator` is reset to 0 on transaction start (`qmgr_initialize_tran_entry`). `NULL_QUERY_ID = 0` is the sentinel.

---

## XASL Cache Invalidation

`qmgr_add_modified_class(thread_p, &class_oid)` appends to the `OID_BLOCK_LIST` on the tran entry. At commit/rollback, `qmgr_clear_relative_cache_entries` iterates these OIDs and calls `xcache_remove_by_oid` to evict XASL plans that referenced the modified class.

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Build mode | `#if defined(CS_MODE) #error` — server (SERVER_MODE or SA_MODE) |
| Mutex | `tran_entry_p->mutex` protects the query entry linked list per transaction |
| Critical section | `CSECT_QPROC_QUERY_TABLE` protects `qmgr_allocate_tran_entries` |
| Memory | Query entries from free list (recycled); temp files from per-type free list; XASL clone from private heap |
| Interrupt check | Executor must call `qmgr_is_query_interrupted` in long scan loops |

---

## Lifecycle

```
Server start:
    qmgr_initialize() → allocate tran entry array

Per connection (transaction):
    qmgr_allocate_tran_entries (if num_trans grew)
    on query execute:
        qmgr_add_query_entry     (QUERY_IN_PROGRESS)
        [execution…]
        qmgr_mark_query_as_completed  (QUERY_COMPLETED)
        xqmgr_end_query          (QUERY_CLOSED → free temp files)
    on tran commit/abort:
        qmgr_clear_trans_wakeup  (free all remaining query entries, temp files)

Server stop:
    qmgr_finalize() → free all
```

---

## Related

- [[components/query-executor]] — `qexec_execute_query` called from `qmgr_process_query`
- [[components/query-cl]] — client-side `qmgr_prepare_query` / `qmgr_execute_query` wrappers (CS_MODE)
- [[components/list-file]] — `QFILE_LIST_ID` produced and cloned here
- [[components/xasl]] — `XASL_CLONE` managed here
- [[components/session]] — session holds holdable cursor list; interacts with `is_holdable` flag
- [[components/query-fetch]] — fetches rows from `QFILE_LIST_ID` after query completes
- [[components/dblink]] — `DBLINK_CONN_ENTRY` managed here
- [[Memory Management Conventions]]
- [[Build Modes (SERVER SA CS)]]
