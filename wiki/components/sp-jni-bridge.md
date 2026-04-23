---
type: component
parent_module: "[[modules/src|src]]"
path: "src/sp/"
status: developing
purpose: "JNI invocation mechanics and DB_VALUE ↔ Java type marshalling across the C/Java process boundary"
key_files:
  - "pl_executor.cpp/hpp — cubpl::executor: invoke/response loop, arg type validation"
  - "pl_signature.cpp/hpp — pl_signature / pl_arg / invoke_java packing"
  - "pl_execution_stack_context.cpp/hpp — execution_stack: send_data_to_java / read_data_from_java"
  - "pl_connection.cpp/hpp — socket send_buffer / receive_buffer"
  - "sp_constants.hpp — SP_CODE_*, METHOD_CALLBACK_*, SP_MODE_ENUM"
tags:
  - component
  - cubrid
  - sp
  - jni
  - marshalling
related:
  - "[[components/sp|sp]]"
  - "[[components/sp-protocol|sp-protocol]]"
  - "[[modules/pl_engine|pl_engine]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# SP JNI Bridge — Invocation Mechanics & Marshalling

> [!key-insight] No in-process JNI
> Despite the historical name "JNI bridge", there is no `jni.h` in `src/sp/`. `cub_pl` runs as a separate OS process. The "bridge" is a custom binary protocol over POSIX sockets.

## Entry Point

`cubpl::executor::execute(DB_VALUE &value)` is the main entry point. It is called from the query executor when an XASL `METHOD_CALL_NODE` is evaluated.

```
executor::execute(DB_VALUE &value)
  ├── change_exec_rights(auth_name)      — elevate to SP definer
  ├── request_invoke_command()           — pack and send invoke request
  │     ├── build session_params delta
  │     ├── prepare_args(args...)        — pack DB_VALUE arguments
  │     └── invoke_java(sig, tc)         — pack metadata
  ├── response_invoke_command(value)     — response loop
  │     ├── read SP_CODE_INTERNAL_JDBC   → response_callback_command()
  │     │       (prepare/execute/fetch/OID/cursor ops back to C)
  │     └── read SP_CODE_RESULT          → response_result(value)
  │               ├── unpack return DB_VALUE
  │               └── unpack OUT arg DB_VALUEs
  └── change_exec_rights(NULL)           — restore rights
```

## Argument Packing

Two objects are packed into a single `cubmem::block` and sent together:

### `invoke_java` (metadata)
Packed fields (in order):
1. `tran_id` (int)
2. `signature` (string: `"ClassName.methodName"`)
3. `auth` (string: definer user name)
4. `lang` (int: `SP_LANG_PLCSQL=0` or `SP_LANG_JAVA=1`)
5. `num_args` (int)
6. For each arg: `arg_mode` (IN=1/OUT=2/INOUT=3), `arg_type` (DB_TYPE)
7. `result_type` (int, DB_TYPE)
8. `transaction_control` (bool: always true for PL/CSQL; configured by `PRM_ID_PL_TRANSACTION_CONTROL` for Java)

### `prepare_args` (values)
Carries the actual `DB_VALUE` array serialized via `cubmethod::dbvalue_java` (from `method_struct_value.hpp`). The format matches what `ExecuteThread.java` expects on the Java side.

## Supported DB_VALUE Types

`executor::is_supported_dbtype()` gates every argument:

| Supported | Unsupported (rejected with `ER_SP_NOT_SUPPORTED_ARG_TYPE`) |
|-----------|------------------------------------------------------------|
| `INTEGER`, `SHORT`, `BIGINT` | `BIT`, `VARBIT` |
| `FLOAT`, `DOUBLE`, `MONETARY`, `NUMERIC` | `BLOB`, `CLOB` |
| `CHAR`, `STRING` | `JSON`, `ENUMERATION` |
| `DATE`, `TIME`, `TIMESTAMP`, `DATETIME` | `TIMESTAMPTZ`, `TIMESTAMPLTZ`, `DATETIMETZ`, `DATETIMELTZ` |
| `SET`, `MULTISET`, `SEQUENCE` | `TABLE` |
| `OID`, `OBJECT`, `RESULTSET`, `NULL` | |

## Response Unpacking

On `SP_CODE_RESULT`:
1. `dbvalue_java::unpack()` reads return value into the caller's `DB_VALUE &value`.
2. If return type is `DB_TYPE_RESULTSET`, the embedded query ID is promoted to a session-scoped cursor via `execution_stack::promote_to_session_cursor()`.
3. All `OUT`/`INOUT` arguments are unpacked in argument order into `m_out_args` vector; callers retrieve them via `executor::get_out_args()`.

On `SP_CODE_ERROR`:
- A Java exception string (stack trace) is unpacked and stored in `execution_stack::m_error_message`.
- Returns `ER_SP_EXECUTE_ERROR` to the query executor.

## Interrupt Handling During Receive

`read_data_from_java()` passes a lambda to `connection::receive_buffer()` with a 500 ms timeout. The lambda calls `execution_stack::interrupt_handler()` which checks `pl_session::is_interrupted()`. If interrupted, returns non-`NO_ERROR` to abort the blocking receive.

## Session Parameter Delta

Before packing `invoke_java`, `executor::request_invoke_command()` calls `session::obtain_session_parameters()`. This compares the set of changed parameter IDs (`m_session_param_changed_ids`) against the last known connection epoch. Only changed parameters are packed in the delta; on a fresh connection (epoch mismatch), all parameters are sent. This keeps the Java-side session context in sync without sending the full parameter list on every call.

## Related

- [[components/sp|sp]] — hub page
- [[components/sp-protocol|sp-protocol]] — wire format details
- [[components/sp-method-dispatch|sp-method-dispatch]] — how the XASL executor triggers `executor::execute()`
- [[modules/pl_engine|pl_engine]] — Java counterpart: `ExecuteThread.java`
