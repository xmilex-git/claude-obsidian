---
created: 2026-04-23
type: component
title: "query-cl — Client-Side Query Entry Points"
parent_module: "[[modules/src|src]]"
path: src/query/query_cl.{c,h}
status: developing
key_files:
  - src/query/query_cl.c
  - src/query/query_cl.h
public_api:
  - prepare_query
  - execute_query
  - prepare_and_execute_query
tags:
  - cubrid
  - query
  - client-side
  - prepare
  - execute
  - xasl
---

# query-cl — Client-Side Query Entry Points

> [!key-insight]
> `query_cl.c` is deliberately tiny (< 200 lines, three public functions). It is the **thin client shim** that bridges `execute_statement.c` (parse-tree layer) to the network interface (`qmgr_prepare_query` / `qmgr_execute_query` in `network_interface_cl.c`). All heavy lifting is done either upstream (optimizer, XASL generation) or downstream (server-side query-manager + executor).

## Purpose

`query_cl.c` provides the three client-side entry points that send XASL byte streams to the server and receive result `QFILE_LIST_ID`s back:

1. **`prepare_query`** — takes a compiled XASL byte stream (`XASL_STREAM`) and registers it with the server's XASL cache. Returns a `XASL_ID` (opaque handle to the cached plan).
2. **`execute_query`** — takes an `XASL_ID` (from a previous prepare) plus host variable values, triggers execution on the server, and returns a `QFILE_LIST_ID` (pointer to the result list file).
3. **`prepare_and_execute_query`** — fused: sends a raw XASL byte stream (no prior cache lookup), gets the result in one round trip. Used by CSQL (direct execution without prepared-statement protocol).

---

## Public Entry Points

| Signature | Role |
|-----------|------|
| `int prepare_query(compile_context* context, xasl_stream* stream)` | Upload XASL stream to server; populate `stream->xasl_id` |
| `int execute_query(const XASL_ID*, QUERY_ID*, int var_cnt, const DB_VALUE* varptr, QFILE_LIST_ID**, QUERY_FLAG, CACHE_TIME*, CACHE_TIME*)` | Execute prepared query; fill `*list_idp` |
| `int prepare_and_execute_query(char* stream, int stream_size, QUERY_ID*, int var_cnt, DB_VALUE* varptr, QFILE_LIST_ID**, QUERY_FLAG)` | One-shot: prepare + execute |

---

## Execution Path

### Prepared Statement (two-phase)

```
SQL text → parser → semantic check → do_prepare_select:
    parser_generate_xasl(parser, stmt)          [XASL tree]
    xts_map_xasl_to_stream(xasl, &stream)       [serialize to bytes]
    prepare_query(context, &stream)
        qmgr_prepare_query(context, stream)     [IPC → server]
            → server: xqmgr_prepare_query       [store in XASL cache, return XASL_ID]
        stream->xasl_id = returned XASL_ID

do_execute_select:
    execute_query(xasl_id, &query_id, var_cnt, varptr, &list_id, flag, …)
        tran_get_query_timeout()
        qmgr_execute_query(xasl_id, …)          [IPC → server]
            → server: xqmgr_execute_query       [lookup XASL cache, run executor, return list_id]
        *list_idp = result
```

### Direct Execution (one-phase)

```
prepare_and_execute_query(stream, size, &qid, var_cnt, varptr, &result, flag)
    if do_Trigger_involved && supplemental_log:
        cdc_Trigger_involved = true
        flag |= TRIGGER_IS_INVOLVED
    tran_get_query_timeout()
    qmgr_prepare_and_execute_query(stream, size, &qid, …)  [IPC → server]
        → server: xqmgr_prepare_and_execute_query
    *result = list_idptr
```

---

## CDC / Trigger Integration

`execute_query` resets `cdc_Trigger_involved = false` before each query (to avoid stale trigger-involvement state from a prior query). `prepare_and_execute_query` conditionally sets it to `true` if `do_Trigger_involved` (set by DML trigger hooks in `execute_statement.c`) and supplemental logging is enabled.

---

## XASL Cache Interaction

```
prepare_query:
    assert(context->sql_hash_text != NULL)    ← SHA1 key must exist
    allocates stream->xasl_id (malloc, caller frees)
    if buffer == NULL and xasl_id is null: cache miss signal
    if recompile flag returned by server: free xasl_id, force recompile
```

If `qo_need_skip_execution()` is true (optimizer debug param), both `prepare_query` and `execute_query` return `NO_ERROR` immediately without contacting the server.

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Build mode | `#if defined(SERVER_MODE) #error` — client only |
| Ownership | `stream->xasl_id` is allocated here (malloc); caller (`do_prepare_select`) must free with `free_and_init` |
| `list_idp` | Returned `QFILE_LIST_ID*` is allocated server-side; caller must free via `QFILE_FREE_AND_INIT_LIST_ID` or `regu_free_listid` |
| Thread safety | Single-threaded per CAS connection; no internal mutexes |
| Timeout | `tran_get_query_timeout()` supplies the millisecond deadline; 0 means no limit |

---

## Lifecycle

```
Per prepared-statement execute cycle:
    prepare_query        ← once per unique SQL text (until cache eviction)
    execute_query        ← once per execute (different host vars each time)
    qmgr_end_query       ← after cursor fully consumed or closed

Per direct-execute cycle:
    prepare_and_execute_query ← one call, one result
    cursor read / qmgr_end_query
```

---

## Related

- [[components/execute-statement]] — `do_prepare_select` / `do_execute_select` call directly into here
- [[components/query-manager]] — server-side `xqmgr_prepare_query` / `xqmgr_execute_query` are the server peers
- [[components/xasl-stream]] — `xts_map_xasl_to_stream` produces the bytes that `prepare_query` uploads
- [[components/xasl]] — `XASL_ID` is the opaque plan handle
- [[components/list-file]] — `QFILE_LIST_ID` is the result handle returned to the client
- [[components/cursor]] — opens cursor over `QFILE_LIST_ID`
- [[components/parser]] — parse tree produced upstream, XASL generated from it
- [[Build Modes (SERVER SA CS)]]
