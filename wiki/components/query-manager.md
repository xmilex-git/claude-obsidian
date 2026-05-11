---
created: 2026-04-23
updated: 2026-05-11
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
related:
  - "[[components/query-executor]]"
  - "[[components/query-fetch]]"
  - "[[components/list-file]]"
  - "[[sources/qp-analysis-tempfile]]"
---

# query-manager — Query Lifecycle and Per-Query Context

> [!key-insight]
> The query manager is the **server-side registry** for in-flight queries. It owns the per-transaction query entry table (`QMGR_QUERY_TABLE`), the temporary file infrastructure (memory-buffered + disk-backed), and the cancellation/interrupt signal path. The actual query execution (`qexec_execute_query`) is **called from inside** `qmgr_process_query` — query-manager wraps executor, not the other way around.

> [!update] 2026-05-11 — Temp file lazy-creation + memory buffer fallback (per [[sources/qp-analysis-tempfile]])
> 본 모듈의 temp-file API 의 의도된 사용 패턴 (분석서 직 정리):
>
> | 함수 | 동작 |
> |---|---|
> | `qmgr_create_new_temp_file` | 실제 임시파일 미생성. **메모리 버퍼 캐시 공간만 할당** (`temp_file_memory_size_in_pages` 만큼). |
> | `qmgr_get_new_page` | 새 page 필요 시 호출. 메모리 버퍼 다 차면 `file_create_temp` → 실제 임시파일 생성 (lazy promotion). |
> | `qmgr_free_list_temp_file` | 메모리 버퍼만 쓰인 경우 메모리만 해제. 임시파일 있으면 제거. |
> | `qmgr_get_old_page` | 기존 file page 읽기. 메모리 버퍼면 메모리 주소, page 면 `pgbuf_fix` 후 page 주소. |
> | `qmgr_create_result_file` | result cache 의 최종 결과용. 메모리 버퍼 우회하고 바로 임시파일 생성 (임시 메모리버퍼 장기 점유 방지 추정). |
>
> ### Tempfile 캐시 (file_tempcache_*)
> 생성/제거 비용 큰 임시파일을 캐시에 보관 후 재사용. `max_entries_in_temp_file_cache`, `max_pages_in_temp_file_cache`.
>
> - `file_tempcache_put` — 캐시로 임시파일 반환
> - `file_tempcache_get` — 캐시에서 임시파일 획득
> - `file_tempcache_push_tran_file` — 임시파일 VPID 를 트랜잭션별 정보에 저장 (commit/rollback 시 삭제 위함)
> - `file_tempcache_pop_tran_file` — 트랜잭션별 정보에서 제거
>
> Trade-off: 캐시된 임시파일이 소유한 page 는 공유 불가 → 영구 임시볼륨 단편화 가능. 그러나 자원 독점 방어 효과.
>
> ### 트랜잭션-단위 정리 (`file_Tempcache.tran_files`)
> 질의 최종 결과 (XASL→list_id) 가 아닌 임시파일은 질의 종료시 모두 제거 (`qexec_clear_xasl → qfile_destroy_list`). 최종 결과 임시파일은 트랜잭션 commit/rollback 시 일괄 제거 (`log_commit_local → file_tempcache_drop_tran_temp_files`). HOLDABLE CURSOR / QUERY RESULT CACHE 는 `file_temp_retire_preserved` 로 commit 이후 연장.
>
> ### Parallel query 시 thread-affinity 제약
> `file_Tempcache.tran_files` 가 **thread-local**. parallel query 에서 다른 thread 가 동일 tran_id 를 쓰면 push/pop 에 락 필요. External sort 는 preserve 패턴으로 우회 (트랜잭션별 정보 미입력). Page FIX/UNFIX 는 동일 thread 권장. private_alloc/private_free 는 동일 thread 에서 free 필요 — `db_change_private_heap(thread_p, 0)` 로 일반 malloc/free 폴백.

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

### Page operations: `tfile` role and minimum context

> [!key-insight] `tfile` is needed only for membuf lookup and free-time page-type discrimination
> The temp-file pointer threaded through `qmgr_get_old_page` / `qmgr_free_old_page` looks essential, but for **disk pages alone** a `vpid` is sufficient on both paths. `tfile` is required only in two situations:
> 1. **Membuf read.** When `vpid->volid == NULL_VOLID`, the encoding makes `vpid->pageid` a membuf array index — the page must be returned via `tfile->membuf[pageid]`. Without `tfile`, the membuf array isn't reachable.
> 2. **Free-time discrimination.** `qmgr_free_old_page` calls `qmgr_get_page_type(page, tfile)` (`query_manager.c:201`) which decides via pointer-range check whether `page` lies in `tfile->membuf[0..membuf_last]`. Buffer-pool pages get `pgbuf_unfix`; membuf pages are no-ops.
>
> When `tfile == NULL`, `qmgr_free_old_page` skips the type check and unconditionally `pgbuf_unfix`'s — fine for disk pages, **wrong** for any membuf page. Likewise, `qmgr_get_old_page` with `tfile == NULL` cannot serve a membuf-encoded VPID.

| Path | `tfile` required? | Why |
|------|-------------------|-----|
| `qmgr_get_old_page` (disk page) | No (could pass NULL) | `pgbuf_fix(vpid)` doesn't read `tfile` |
| `qmgr_get_old_page` (membuf page, `volid == NULL_VOLID`) | **Yes** | `tfile->membuf[pageid]` lookup |
| `qmgr_free_old_page` (disk page) | No-but-conventional | Always `pgbuf_unfix`; `qmgr_get_page_type` returns `TEMP_FILE_PAGE` regardless of which `tfile` |
| `qmgr_free_old_page` (membuf page) | **Yes** | Need to detect "do nothing" via membuf pointer range |
| `qmgr_set_dirty_page` / `qmgr_get_new_page` | (write path; not on read-only scans) | — |

This is why parallel scanners that distribute disk pages across workers (parallel hash join split phase, parallel heap scan, parallel index build) **only need to track the owning tfile per sector** — not for the disk-page read/free itself, but for two reasons: handling membuf pages (single-owner CAS-claim) and for API contract when calling `qfile_*` helpers that internally consult `tfile_vfid` (e.g. overflow-chain traversal). See [[sources/2026-04-28-tfile-role-analysis]] for the full analysis.

A dependent list's `tfile_vfid` always has `membuf == NULL` (asserted by `qfile_connect_list` — see [[components/list-file]]), so the membuf branch never fires for non-base pages.

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
