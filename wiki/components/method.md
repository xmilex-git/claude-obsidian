---
type: component
parent_module: "[[modules/src|src]]"
path: "src/method/"
status: active
purpose: "Method and stored-procedure invocation during query execution: scan-time dispatch via S_METHOD_SCAN, builtin C object methods, and the shared method_invoke_group bridge to the PL engine"
key_files:
  - "method_scan.cpp/hpp — cubscan::method::scanner; S_METHOD_SCAN integration"
  - "query_method.cpp/hpp — method_dispatch (CS/SA entry), method_invoke_builtin, vobj fixup"
  - "method_invoke_group.hpp (in src/sp/) — cubmethod::method_invoke_group; shared with src/sp/"
  - "method_struct_invoke.cpp/hpp — header + prepare_args packable objects"
  - "method_struct_value.cpp/hpp — DB_VALUE packing for method arguments/results"
  - "method_struct_oid_info.cpp/hpp — OID info structures for instance method dispatch"
  - "method_struct_parameter_info.hpp — db_parameter_info"
  - "method_struct_query.cpp/hpp — query structures (prepare_info, execute_info, column_info)"
  - "method_struct_schema_info.cpp/hpp — schema info structures"
  - "method_query_handler.cpp/hpp — cubmethod::query_handler; server-side JDBC callbacks"
  - "method_query_result.cpp/hpp — query_result, query_result_info"
  - "method_query_util.cpp/hpp — utility helpers"
  - "method_schema_info.cpp/hpp — schema information handling"
  - "method_oid_handler.cpp/hpp — OID object get/put/cmd callbacks"
  - "method_callback.cpp/hpp — cubmethod::callback_handler; CAS-callback dispatch"
  - "method_error.cpp/hpp — error_context"
public_api:
  - "cubscan::method::scanner::init(thread_p, sig_array, list_id) → int"
  - "cubscan::method::scanner::open() → int"
  - "cubscan::method::scanner::next_scan(val_list_node &) → SCAN_CODE"
  - "cubscan::method::scanner::close() → int"
  - "method_dispatch(rc, methoddata, size) → int   [CS_MODE]"
  - "method_dispatch(unpacker &) → int              [SA_MODE]"
  - "method_error(rc, error_id) → int              [CS_MODE]"
  - "cubmethod::get_callback_handler() → callback_handler*"
tags:
  - component
  - cubrid
  - method
  - scan
  - query
related:
  - "[[modules/src|src]]"
  - "[[components/sp|sp]]"
  - "[[components/sp-method-dispatch|sp-method-dispatch]]"
  - "[[components/sp-jni-bridge|sp-jni-bridge]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/method-invoke-group|method-invoke-group]]"
  - "[[components/method-scan|method-scan]]"
  - "[[modules/pl_engine|pl_engine]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/method/` — Method & SP Invocation from Queries

`src/method/` is the CUBRID subsystem that handles invocation of **instance/class methods** (CUBRID's legacy OO feature) and **stored procedures** at query-scan time. It owns the `S_METHOD_SCAN` scan type consumed by [[components/scan-manager|scan-manager]] and the [[components/query-executor|query-executor]], plus the client-side callback machinery needed when a builtin C method or SP triggers server-side JDBC during scan execution.

## Why This Directory Exists

CUBRID inherited a Smalltalk/OO-era feature: tables can have C-language methods attached to classes (distinct from Java stored procedures). A query such as:

```sql
SELECT o.my_method(arg) FROM some_class o
```

causes the query engine to invoke `my_method` for each row — a scan-time call, not a pre-query call. This is architecturally different from `CALL my_proc()` (which is a top-level SQL statement handled by [[components/sp|src/sp/]]).

Over time the Java SP machinery also routes through this layer's shared `method_invoke_group` dispatch struct (owned in `src/sp/`), making `src/method/` a **shared execution path** for both older builtin C methods and modern Java SPs when they appear inside a query scan.

## Architecture Overview

```
SQL: SELECT obj.c_method(args) FROM tbl
         │
         │  XASL compiled with CSELECT / method scan list
         ▼
  query_executor.c
    → scan_open_method_scan()          [scan_manager.c]
      → cubscan::method::scanner::init()   [method_scan.cpp]
        → new cubmethod::method_invoke_group(sig_array)
    → scan_next_method_scan()
      → cubscan::method::scanner::next_scan()
        1. get_single_tuple()          ← reads arg tuple from qfile list
        2. method_invoke_group::execute(args)
              │
              ├─ Builtin C method → method_dispatch() → obj_send_array()
              │                     (CS_MODE: sends to server via net)
              │                     (SA_MODE: calls in-process)
              │
              └─ Java method/SP  → cubpl::execution_stack → cub_pl process
                                   (see components/sp)
        3. Clone return DB_VALUEs into val_list_node for query engine
```

## Key Structural Split by Build Mode

`src/method/` files are aggressively guarded by build mode:

| File / class | SERVER_MODE | SA_MODE | CS_MODE |
|---|---|---|---|
| `method_scan.hpp` (scanner) | yes | yes | no |
| `method_invoke_group.hpp` | yes | yes | no |
| `query_method.cpp` — `method_dispatch()` | no | yes | yes |
| `method_callback.hpp` (callback_handler) | no | yes | yes |
| `method_query_handler.hpp` (query_handler) | no | yes | yes |

The server side owns the scanner and `method_invoke_group`. The client side (CAS process in CS_MODE, or in-process in SA_MODE) handles callbacks from the invoked method back into the query engine (prepare/execute/fetch SQL, OID operations, schema info).

## Methods vs. Stored Procedures

| Aspect | Builtin C Method | Java SP / PL-CSQL |
|---|---|---|
| Definition | `CREATE METHOD` attached to a class | `CREATE PROCEDURE / FUNCTION` |
| Language | C (compiled into cub_server / libcubrid) | Java / PL-CSQL (runs in `cub_pl`) |
| Invocation site | Scan time inside a SELECT | `CALL stmt` or scan time |
| Object required | Yes — first argument is the target OID | No |
| Dispatch path | `obj_send_array()` via `object_accessor` | `cubpl::executor::execute()` via socket |
| Recursion guard | `METHOD_MAX_RECURSION_DEPTH` | Same constant |
| Auth context | `AU_ENABLE` then `AU_DISABLE` around call | `change_exec_rights(auth_name)` |

## `method_invoke_group` — Shared Dispatch Struct

`cubmethod::method_invoke_group` (declared in `src/sp/method_invoke_group.hpp`, used by both `src/method/` and `src/sp/`) batches all methods called together in one scan step:

```cpp
class method_invoke_group {
  void begin();                // claim pl_connection from pool
  int  prepare(args);         // pack + send METHOD_REQUEST_ARG_PREPARE
  int  execute(args);         // send invoke request; run response loop
  int  reset(bool is_end_query);
  void end();                  // release connection

  DB_VALUE &get_return_value(int index);
  std::string get_error_msg();
};
```

One `method_invoke_group` is created per `scanner::init()` call (one per `S_METHOD_SCAN` node). It holds a `cubpl::execution_stack*` and owns the result `DB_VALUE` vector. See [[components/method-invoke-group|method-invoke-group]].

## Callback Handler (`method_callback.cpp`)

When a Java SP or builtin method issues a JDBC query back into the engine (server-side JDBC), the CAS process receives a `METHOD_REQUEST_CALLBACK` and routes it to `cubmethod::callback_handler::callback_dispatch()`:

| Callback command | Handler |
|---|---|
| `prepare` / `execute` / `fetch` | `query_handler::prepare()` / `execute()` |
| `oid_get` / `oid_put` / `oid_cmd` | `oid_handler` |
| `collection_cmd` | `oid_handler` |
| `end_transaction` | `transaction_cl` commit/abort |
| `change_rights` | `authenticate` privilege switch |
| `get_sql_semantics` / `get_global_semantics` | PL/CSQL compiler support |

`callback_handler` maintains a `std::vector<query_handler*>` (CAS-side statement cache), `m_sql_handler_map` (SQL→handler-id), and `m_qid_handler_map` (query-id→handler-id).

## vobj Fixup

Because method arguments may travel as raw OIDs or virtual objects (`DB_TYPE_OID`, `DB_TYPE_VOBJ`) from the server to the CAS, `method_prepare_arguments()` calls `method_fixup_vobjs()` on each argument before storing them in `runtime_args`. This converts OIDs to `DB_TYPE_OBJECT` handles and VOBJs to vmops so that `obj_send_array()` can dereference them correctly. This is explicitly noted as a port from `cursor.c`.

## Scan-Time Performance Gotchas

- Each row in the scan triggers `next_scan()` → `method_invoke_group::execute()`. For Java methods this involves a socket round-trip per row — expensive at scale.
- `m_arg_vector` and `m_arg_dom_vector` are allocated once in `init()` and reused per row; `db_value_clear` + `db_make_null` on each iteration avoids per-row allocation.
- `m_dbval_list` result pointers are reallocated each `next_scan()` call (`db_private_alloc` per return value) and cloned from `m_result_vector`; the query executor owns these until the next call.

## Sub-Components

- [[components/method-invoke-group|method-invoke-group]] — `cubmethod::method_invoke_group`: shared dispatch struct, execution-stack binding, result vector
- [[components/method-scan|method-scan]] — `cubscan::method::scanner`: `S_METHOD_SCAN` integration with scan-manager

## Related

- Parent: [[modules/src|src]]
- [[components/sp|sp]] — owns `cub_pl` lifecycle; Java SPs route here from scanner
- [[components/sp-method-dispatch|sp-method-dispatch]] — how METHOD_CALL_NODE in XASL reaches executor
- [[components/scan-manager|scan-manager]] — contains `METHOD_SCAN_ID msid` in `SCAN_ID` union; dispatches to `scan_next_method_scan()`
- [[components/query-executor|query-executor]] — opens and iterates `S_METHOD_SCAN`
- [[modules/pl_engine|pl_engine]] — Java counterpart that executes SP/PL-CSQL code
- [[Build Modes (SERVER SA CS)]] — determines which half of `query_method.cpp` compiles
- Source: [[sources/cubrid-src-method|cubrid-src-method]]
