---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/"
status: active
purpose: "Server-side XASL execution engine: query executor, scan managers, list files, aggregation, analytic functions, partition pruning, dblink, memoization, and filter predicate cache"
key_files:
  - "query_executor.c (~27K lines — qexec_execute_mainblock entry point)"
  - "scan_manager.c (scan_open_*/scan_next_scan dispatch for all scan types)"
  - "fetch.c (REGU_VARIABLE evaluation, tuple fetching)"
  - "query_manager.c (query cache, temp file lifecycle, result sets)"
  - "list_file.c (qfile_* — intermediate/result temp list files + sorting)"
  - "string_opfunc.c (~28K lines — CONCAT, SUBSTR, etc.)"
  - "arithmetic.c (numeric/date/time function implementations)"
  - "query_opfunc.c (aggregate function evaluation)"
  - "query_analytic.cpp (window/OVER clause execution)"
  - "query_hash_scan.c (hash join scan)"
  - "partition_sr.c (runtime partition pruning)"
  - "dblink_scan.c + dblink_2pc.c (remote query & 2-phase commit)"
  - "filter_pred_cache.c (fpcache_* — predicate expression cache)"
  - "memoize.hpp / memoize.cpp (result memoization)"
  - "xasl_to_stream.c (client-side: XASL → byte stream)"
  - "stream_to_xasl.c (server-side: byte stream → XASL)"
  - "vacuum.c (MVCC garbage collection)"
public_api:
  - "qexec_execute_query(thread_p, xasl, dbval_cnt, dbval_ptr, query_id) → QFILE_LIST_ID*"
  - "qexec_execute_mainblock(thread_p, xasl, xstate, lock_info) → int"
  - "scan_open_heap_scan / scan_open_index_scan / scan_open_list_scan / ..."
  - "scan_next_scan(thread_p, scan_id) → SCAN_CODE"
  - "qfile_open_list / qfile_add_tuple_to_list / qfile_sort_list"
  - "fpcache_claim / fpcache_retire"
  - "memoize_get / memoize_put / memoize_put_nullptr"
tags:
  - component
  - cubrid
  - query
  - xasl
  - server
related:
  - "[[modules/src|src]]"
  - "[[Query Processing Pipeline]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/cursor|cursor]]"
  - "[[components/partition-pruning|partition-pruning]]"
  - "[[components/dblink|dblink]]"
  - "[[components/list-file|list-file]]"
  - "[[components/aggregate-analytic|aggregate-analytic]]"
  - "[[components/filter-pred-cache|filter-pred-cache]]"
  - "[[components/memoize|memoize]]"
  - "[[components/parser|parser]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/query/` — XASL Execution Layer

Server-side query execution for CUBRID. Receives deserialized `XASL_NODE` trees and drives them to completion, producing result `QFILE_LIST_ID`s consumed by the client-side cursor.

> [!key-insight] Client/server split
> Everything in this directory (except `xasl_to_stream.c` and `cursor.c`) runs **server-side**, guarded by `#if !defined(SERVER_MODE) && !defined(SA_MODE)`. The parser and optimizer run on the client; the server only sees a serialized XASL byte stream.

## Architecture Overview

```
Client: SQL → Parser → Optimizer → XASL_NODE tree
              ↓
        xasl_to_stream.c  (serialize to byte stream)
              ↓ network ↓
        stream_to_xasl.c  (deserialize on server)
              ↓
        qexec_execute_query()
              ↓
        qexec_execute_mainblock()   ← main tree walker (~27K lines)
              ↓
        For each XASL_NODE type:
          BUILDLIST_PROC  → scan + materialize to list file
          BUILDVALUE_PROC → single-row aggregation
          UNION / DIFFERENCE / INTERSECTION → set operations
          UPDATE_PROC / DELETE_PROC / INSERT_PROC → DML execution
              ↓
        scan_manager.c   ← scan dispatch
          heap scan / parallel heap scan
          index scan (btree, covering index, ISS, MRO)
          list scan (from list file)
          hash scan (hash join probe)
          set scan / JSON table scan / method scan / dblink scan
              ↓
        fetch.c          ← REGU_VARIABLE evaluation (recursive)
              ↓
        list_file.c      ← intermediate result materialization & sort
              ↓
        qfile_list_id    → returned to client cursor
```

## Key Data Types

| Type | Purpose |
|------|---------|
| `XASL_NODE` | Executable plan node (SELECT, UPDATE, INSERT, etc.) |
| `XASL_STATE` | Per-query execution state; holds `VAL_DESCR`, `QUERY_ID`, error line |
| `VAL_DESCR` | Value descriptor: DB_VALUE array, sys datetime, random seeds |
| `PRED_EXPR` | Predicate expression tree (WHERE/HAVING) — `cubxasl::pred_expr` |
| `REGU_VARIABLE` | Register variable — column ref, constant, or sub-expression |
| `VAL_LIST` | Current-tuple value list |
| `QFILE_LIST_ID` | Temp list file handle for intermediate/final results |
| `SCAN_ID` | Unified scan state (union of all scan types) |

## Top-Level Subsystems

| Sub-component | Page | Primary files |
|---------------|------|---------------|
| XASL executor | [[components/query-executor]] | `query_executor.c`, `query_manager.c` |
| Scan manager | [[components/scan-manager]] | `scan_manager.c` |
| Cursor (client) | [[components/cursor]] | `cursor.c` |
| Partition pruning | [[components/partition-pruning]] | `partition_sr.c`, `partition.c` |
| DBLink | [[components/dblink]] | `dblink_scan.c`, `dblink_2pc.c` |
| List files | [[components/list-file]] | `list_file.c` |
| Aggregate + analytic | [[components/aggregate-analytic]] | `query_aggregate.hpp`, `query_analytic.hpp`, `query_opfunc.c` |
| Filter pred cache | [[components/filter-pred-cache]] | `filter_pred_cache.c` |
| Memoization | [[components/memoize]] | `memoize.hpp`, `memoize.cpp` |
| Parallel execution | [[components/parallel-query]] | `parallel/` (separately ingested) |

## Entry Points

### `qexec_execute_query`
Top-level call from server interface. Looks up or creates query state, then calls `qexec_execute_mainblock`. Returns `QFILE_LIST_ID*` — the final result spool.

### `qexec_execute_mainblock`
The XASL tree walker. Dispatches on `xasl->type`:
- `BUILDLIST_PROC` — opens scan(s), iterates tuples, materializes to list file. Calls group-by sort, analytic window evaluation, hash aggregate when applicable.
- `BUILDVALUE_PROC` — runs aggregate over single result row.
- `UNION_PROC` / `DIFFERENCE_PROC` / `INTERSECTION_PROC` — set-operation combining of list files.
- `UPDATE_PROC` / `DELETE_PROC` / `INSERT_PROC` — DML paths (lock acquisition, heap update/delete/insert, index maintenance).

### `qexec_execute_subquery_for_result_cache`
Separate entry for uncorrelated subqueries used with the result cache.

## XASL Serialization

> [!warning] Exact byte-stream match required
> `xasl_to_stream.c` (client) and `stream_to_xasl.c` (server) must agree exactly. A version mismatch between client and server causes crashes or silent corruption. The stream is versioned but this is easy to break when adding fields to `XASL_NODE` or `REGU_VARIABLE`.

## Conventions

- All server functions take `THREAD_ENTRY *thread_p` as first parameter.
- Function prefixes: `qexec_` (executor), `scan_` (scan manager), `qfile_` (list files), `qdata_` (aggregate/analytic data ops), `fpcache_` (filter pred cache), `memoize_` (memoize).
- Aggregate state tracked in `AGGREGATE_TYPE` linked list on the XASL node.
- All function implementations receive `DB_VALUE *` args, write result to `DB_VALUE *`.
- `GOTO_EXIT_ON_ERROR` macro is the standard error-path idiom in `query_executor.c`.

## Function-Adding Guideline

To add a new built-in SQL function, the change spans four modules:

```
src/parser/ → type_checking.c → xasl_generation.c → src/query/
```

Implement in `string_opfunc.c` or `arithmetic.c` and wire into `fetch.c`. See [[components/parser]] for the parser side.

## Related

- Parent: [[modules/src|src]]
- [[Query Processing Pipeline]] — end-to-end query path
- [[components/parallel-query|parallel-query]] — parallel extensions (heap scan, hash join, sort, subquery)
- [[components/parser|parser]] — upstream: SQL → XASL generation
- Source: [[sources/cubrid-src-query|cubrid-src-query]]
