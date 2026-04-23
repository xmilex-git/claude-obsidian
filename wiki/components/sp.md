---
type: component
parent_module: "[[modules/src|src]]"
path: "src/sp/"
status: developing
purpose: "Bridge between C++ engine and the Java PL engine: conveys SQL CALL/method invocations across the process boundary, manages the PL server lifecycle, and marshalls DB_VALUE ↔ Java types"
key_files:
  - "sp_catalog.cpp/hpp — SP DDL: create/drop/alter stored procedures in _db_stored_procedure"
  - "sp_constants.hpp — SP_TYPE_ENUM, SP_MODE_ENUM, SP_LANG_ENUM, METHOD_CALLBACK_* opcodes"
  - "pl_executor.cpp/hpp — core invoke/response loop (cubpl::executor)"
  - "pl_session.cpp/hpp — per-session state: execution stack, connection views, cursors"
  - "pl_execution_stack_context.cpp/hpp — one execution_stack per recursive SP call"
  - "pl_signature.cpp/hpp — pl_signature (name, auth, arg modes/types, result_type)"
  - "pl_connection.cpp/hpp — connection_pool (size 10) + connection RAII view"
  - "pl_comm.c/h — raw socket I/O: UDS (primary, non-Windows) or TCP; ping, read/write wrappers"
  - "pl_sr.cpp/h — server_manager: forks cub_pl, monitors health, exposes get_connection_pool()"
  - "pl_sr_jvm.h — thin shim pl_start_jvm_server / pl_server_port"
  - "pl_file.c/h — PL info file read/write (port, pid) for inter-process discovery"
  - "jsp_cl.cpp/h — client-side SP API (CS_MODE)"
  - "pl_query_cursor.cpp/hpp — server-side query cursor promoted into session scope"
  - "pl_compile_handler.cpp/hpp / pl_struct_compile.cpp/hpp — compile handshake with PL engine"
public_api:
  - "pl_server_init(db_name) — called at boot; forks cub_pl if not running"
  - "pl_server_destroy() — tears down connection pool and monitor daemon"
  - "pl_server_wait_for_ready() — blocks until cub_pl is accepting connections"
  - "get_connection_pool() — returns PL_CONNECTION_POOL* for use by executor"
  - "cubpl::executor::fetch_args_peek(...) — binds regu-variable or DB_VALUE args"
  - "cubpl::executor::execute(DB_VALUE &value) — full SP call: invoke + response loop"
  - "cubpl::executor::get_out_args() — retrieves OUT/INOUT values after execute()"
  - "sp_add_stored_procedure(SP_INFO &) — DDL: write row to _db_stored_procedure"
tags:
  - component
  - cubrid
  - sp
  - jni
  - stored-procedure
related:
  - "[[modules/src|src]]"
  - "[[modules/pl_engine|pl_engine]]"
  - "[[components/sp-jni-bridge|sp-jni-bridge]]"
  - "[[components/sp-method-dispatch|sp-method-dispatch]]"
  - "[[components/sp-protocol|sp-protocol]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/authenticate|authenticate]]"
  - "[[components/system-catalog|system-catalog]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/sp/` — Stored Procedure JNI Bridge

`src/sp/` is the C++-side bridge that lets the CUBRID engine execute Java/PL-CSQL stored procedures hosted in a separate `cub_pl` JVM process. It owns:

1. **Catalog DDL** — storing SP definitions in system tables
2. **Process lifecycle** — forking, monitoring, and bootstrapping `cub_pl`
3. **Connection pool** — pooled socket connections to `cub_pl`
4. **Execution orchestration** — argument marshalling, invoke/response protocol, callback servicing
5. **Session management** — per-session execution stacks and result-set cursors

## Architecture Overview

```
SQL: CALL my_proc(arg1, arg2)
         │
         │  XASL (METHOD_CALL_NODE) evaluated by
         ▼
  components/query-executor  ──→  cubpl::executor::fetch_args_peek()
                                           │
                                           │  pl_signature (name, auth, arg modes/types)
                                           ▼
                                  cubpl::executor::execute()
                                           │
                              ┌────────────┴────────────┐
                              │  execution_stack         │
                              │  (one per recursive call)│
                              └────────────┬────────────┘
                                           │  SP_CODE_INVOKE  (packed binary)
                                           ▼
                              pl_connection (UDS or TCP socket)
                                           │
                                     cub_pl process
                                  (pl_engine/ — Java)
                                           │
                              ┌────────────┴──────────────────────┐
                              │  SP_CODE_INTERNAL_JDBC callbacks   │
                              │  (prepare, execute, fetch, …)     │
                              │  or SP_CODE_RESULT / SP_CODE_ERROR │
                              └────────────┬──────────────────────┘
                                           │
                                  executor::response_invoke_command()
                                  ← unpacks DB_VALUE return + OUT args
```

All engine-side code is gated by `#if !defined(SERVER_MODE) && !defined(SA_MODE) → #error`. The client-side API (`jsp_cl.cpp`) is for `CS_MODE` only.

## JVM Lifecycle

`pl_sr.cpp` implements `cubpl::server_manager` (singleton, created at `pl_server_init`):

| Step | What happens |
|------|-------------|
| `pl_server_init()` | Constructs `server_manager`, creates `connection_pool(10)`, spawns `"pl-monitor"` daemon (SERVER_MODE) or runs `do_monitor()` synchronously (SA_MODE) |
| `do_monitor()` | Calls `create_child_process("cub_pl", db_name)`. On success, sleeps 1 s then sets state `READY_TO_INITIALIZE` |
| `do_initialize()` | Polls socket (up to 10 tries) until `cub_pl` is connectable, then sends `bootstrap_request` (contains all relevant system parameters) |
| `pl_server_wait_for_ready()` | In SERVER_MODE: `condition_variable::wait` until state == `RUNNING`; in SA_MODE: retry loop (10 tries) |
| Monitor loop | Daemon fires every 1 s. Pings `cub_pl`; if dead, increments epoch (invalidates stale connections) and re-forks |
| `pl_server_destroy()` | Destroys daemon, deletes `connection_pool` |

> [!key-insight] JVM is a separate OS process
> There is no in-process JNI. The "JNI bridge" is a misnomer historically — current architecture forks a separate `cub_pl` binary that hosts the JVM. Communication is exclusively via sockets.

## Threading Model

- **Server side**: each SQL worker thread that executes an SP call gets one `execution_stack`. In a recursive SP call, each level uses a fresh `cubthread::entry` worker and its own `execution_stack`. The `pl_session` tracks all stacks via `exec_stack_map_type`.
- **Connection pool**: `connection_pool` holds 10 socket connections in a `std::queue` guarded by `std::mutex`. `claim()` blocks until one is available; `retire()` returns it. The pool uses an atomic `m_epoch`; connections with stale epoch reconnect on the next use.
- **Interrupt**: `pl_session::set_interrupt()` stores an interrupt code + message. The receive loop (`read_data_from_java`) checks it via a lambda (`interrupt_handler`) passed to `connection::receive_buffer` with a 500 ms timeout.

## Call Protocol (SP_CODE opcodes)

Defined in `pl_comm.h` and `sp_constants.hpp`:

| Opcode | Direction | Meaning |
|--------|-----------|---------|
| `SP_CODE_INVOKE` (0x01) | C → Java | Start SP execution; carries `invoke_java` + `prepare_args` packed objects |
| `SP_CODE_RESULT` (0x02) | Java → C | SP returned normally; payload = packed `DB_VALUE` return + OUT args |
| `SP_CODE_ERROR` (0x04) | Java → C | SP threw an exception; payload = error string |
| `SP_CODE_INTERNAL_JDBC` (0x08) | Java → C | Callback: Java needs C engine for query prepare/execute/fetch/OID/cursor/transaction |
| `SP_CODE_DESTROY` (0x10) | C → Java | Shut down this connection |
| `SP_CODE_COMPILE` (0x80) | C → Java | Send PL/CSQL source for compilation |
| `SP_CODE_UTIL_PING` (0xDE) | C → Java | Health check |
| `SP_CODE_UTIL_BOOTSTRAP` (0xDD) | C → Java | Send initial session parameters |

The response loop in `executor::response_invoke_command()` is a `do-while` that processes `SP_CODE_INTERNAL_JDBC` callbacks until it receives `SP_CODE_RESULT` or `SP_CODE_ERROR`.

## Argument Marshalling

`pl_signature` carries the full SP contract:

```cpp
struct pl_signature {
  int type;         // PL_TYPE_JAVA_SP | PL_TYPE_PLCSQL | PL_TYPE_INSTANCE_METHOD
  char *name;       // canonical SP name
  char *auth;       // definer (RIGHTS OWNER) or invoker name
  int result_type;  // DB_TYPE
  pl_arg arg;       // arg_size, arg_mode[], arg_type[]
  pl_ext ext;       // .sp.target_class_name / .sp.target_method_name
};
```

`invoke_java::pack()` writes: `tran_id, signature ("class.method"), auth, lang, num_args, [arg_mode, arg_type]*, result_type, transaction_control`.

`prepare_args` carries the actual `DB_VALUE` payload alongside `invoke_java`; both are packed into a single `cubmem::block` sent via `send_data_to_java()`.

Unsupported types (`DB_TYPE_BLOB`, `DB_TYPE_CLOB`, `DB_TYPE_JSON`, `DB_TYPE_BIT`, timezone-aware timestamps, `DB_TYPE_ENUMERATION`) are rejected by `executor::is_supported_dbtype()` before the socket write.

## Error Propagation

Java exceptions surface as `SP_CODE_ERROR` packets containing a Java stack trace string. `executor::response_result()` calls `m_stack->set_error_message(error_msg)` and returns `ER_SP_EXECUTE_ERROR`. The C caller then reads the message via `er_msg()` or `execution_stack::get_error_message()`.

Network errors set `ER_SP_NETWORK_ERROR`. Connection failures set `ER_SP_CANNOT_CONNECT_PL_SERVER`. All use `er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, …)` — standard C error model, no C++ exceptions.

## SP Catalog

System tables (defined in `sp_constants.hpp`):

| Table | Purpose |
|-------|---------|
| `_db_stored_procedure` | One row per SP: name, type, return type, arg count, lang, target_class/method, owner, directives, timestamps |
| `_db_stored_procedure_args` | One row per argument: name, mode, type, default value |
| `_db_stored_procedure_code` | Source code and compiled bytecode (JAR) |

`sp_catalog.cpp` provides `sp_add_stored_procedure()`, `sp_add_stored_procedure_argument()`, `sp_add_stored_procedure_code()`, and `sp_edit_stored_procedure_code()`.

`sp_directive` flags: `SP_DIRECTIVE_RIGHTS_OWNER` (0x00, default) vs `SP_DIRECTIVE_RIGHTS_CALLER` (0x01); `SP_DIRECTIVE_DETERMINISTIC` (0x02).

## Sub-Components

- [[components/sp-jni-bridge|sp-jni-bridge]] — JNI invocation mechanics and argument marshalling
- [[components/sp-method-dispatch|sp-method-dispatch]] — how methods/SPs are dispatched from queries
- [[components/sp-protocol|sp-protocol]] — wire protocol between C engine and `cub_pl`

## Execution Rights

Before invoking, `executor::execute()` calls `change_exec_rights(auth_name)`, which sends `METHOD_CALLBACK_CHANGE_RIGHTS` to the CAS client process (via `execution_stack::send_data_to_client()`). This temporarily elevates the session to the SP definer's rights (for `RIGHTS OWNER` SPs). On return or error the rights are restored with `change_exec_rights(NULL)`.

## Related

- Parent: [[modules/src|src]]
- [[modules/pl_engine|pl_engine]] — Java counterpart (`ExecuteThread.java`, JVM hosting)
- [[components/query-executor|query-executor]] — initiates SP calls during XASL evaluation
- [[components/authenticate|authenticate]] — execution-rights stack consulted by `change_exec_rights`
- [[components/system-catalog|system-catalog]] — `_db_stored_procedure` tables live here
- [[components/method]] — method invocation (forward-ref; to be created)
- [[Build Modes (SERVER SA CS)]]
- Source: [[sources/cubrid-src-sp|cubrid-src-sp]]
