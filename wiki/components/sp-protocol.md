---
type: component
parent_module: "[[modules/src|src]]"
path: "src/sp/"
status: developing
purpose: "Wire protocol between cub_server/cub_pl: transport selection, framing, opcode set, callback round-trips"
key_files:
  - "pl_comm.c/h — raw socket I/O: pl_connect_server, pl_writen, pl_readn, pl_readn_with_timeout"
  - "pl_connection.cpp/hpp — connection / connection_pool RAII layer"
  - "pl_execution_stack_context.cpp/hpp — send_data_to_java / read_data_from_java"
  - "pl_file.c/h — PL info file: port number and PID discovery"
  - "sp_constants.hpp — SP_CODE_* opcodes, METHOD_CALLBACK_* codes"
  - "pl_comm.h — SP_CODE enum definition"
tags:
  - component
  - cubrid
  - sp
  - protocol
  - socket
related:
  - "[[components/sp|sp]]"
  - "[[components/sp-jni-bridge|sp-jni-bridge]]"
  - "[[modules/pl_engine|pl_engine]]"
created: 2026-04-23
updated: 2026-04-23
---

# SP Protocol — Transport & Wire Format

## Transport Selection

`pl_comm.c::pl_connect_server()` selects the transport at connect time:

| Platform | `server_port` value | Transport |
|----------|--------------------|-----------| 
| Linux/macOS | `PL_PORT_UDS_MODE` | Unix domain socket (UDS) — `pl_connect_server_uds()` uses `CUBRID_DATABASES/<db>/cub_pl.sock` |
| Linux/macOS | any positive port | TCP loopback — `pl_connect_server_tcp()` |
| Windows | any | TCP only (`PL_PORT_UDS_MODE` not available) |

The port number is discovered from the PL info file written by `cub_pl` at startup (`pl_file.c::pl_read_info()`). On UDS-capable platforms, if the info file shows `PL_PORT_UDS_MODE`, the connection uses the socket file path returned by `pl_get_socket_file_path(db_name)`.

## Framing

There is no explicit length-prefix frame header in `pl_comm.c` primitives (`pl_writen` / `pl_readn`). Framing is provided at the `cubmem::block` / `cubpacking::packer` layer:

- `connection::send_buffer(const cubmem::block &mem)` — writes the block (includes its own size header from `pack_data_block`)
- `connection::receive_buffer(cubmem::block &b, ...)` — reads into a `cubmem::block`; the `receive_buffer` with timeout polls in 500 ms intervals calling the interrupt lambda between polls

## Opcode Set

Full set from `pl_comm.h` and `sp_constants.hpp`:

### Primary SP_CODE (C ↔ Java top-level messages)

| Code | Hex | Direction | Payload |
|------|-----|-----------|---------|
| `SP_CODE_INVOKE` | 0x01 | C → Java | `invoke_java` + `prepare_args` (session params delta + arg DB_VALUEs) |
| `SP_CODE_RESULT` | 0x02 | Java → C | Return `DB_VALUE` + OUT arg `DB_VALUE`s |
| `SP_CODE_ERROR` | 0x04 | Java → C | UTF-8 error/stack-trace string |
| `SP_CODE_INTERNAL_JDBC` | 0x08 | Java → C | Callback opcode; inner payload is `METHOD_CALLBACK_*` |
| `SP_CODE_DESTROY` | 0x10 | C → Java | Teardown |
| `SP_CODE_COMPILE` | 0x80 | C → Java | PL/CSQL source for compile; response is compile result |
| `SP_CODE_UTIL_BOOTSTRAP` | 0xDD | C → Java | Server init: send all session parameters |
| `SP_CODE_UTIL_PING` | 0xDE | C → Java | Keepalive; Java echoes back |
| `SP_CODE_UTIL_STATUS` | 0xEE | C → Java | Status query |

### METHOD_CALLBACK_* (inside SP_CODE_INTERNAL_JDBC)

Java-to-C callback codes (Java needs the C engine for database operations during SP execution):

| Code | Value | Operation |
|------|-------|-----------|
| `METHOD_CALLBACK_END_TRANSACTION` | 1 | COMMIT / ROLLBACK |
| `METHOD_CALLBACK_QUERY_PREPARE` | 2 | Prepare a SQL statement |
| `METHOD_CALLBACK_QUERY_EXECUTE` | 3 | Execute a prepared statement |
| `METHOD_CALLBACK_GET_DB_PARAMETER` | 4 | Read a system parameter |
| `METHOD_CALLBACK_FETCH` | 8 | Fetch rows from a cursor |
| `METHOD_CALLBACK_OID_GET` | 10 | Get OID attributes |
| `METHOD_CALLBACK_OID_PUT` | 11 | Set OID attributes |
| `METHOD_CALLBACK_OID_CMD` | 17 | OID operation |
| `METHOD_CALLBACK_COLLECTION` | 18 | Collection operation |
| `METHOD_CALLBACK_MAKE_OUT_RS` | 33 | Promote cursor to OUT RESULTSET |
| `METHOD_CALLBACK_GET_GENERATED_KEYS` | 34 | Retrieve auto-generated keys |
| `METHOD_CALLBACK_SET_PL_SESSION_PARAM` | 50 | Java → C parameter sync |
| `METHOD_CALLBACK_GET_SQL_SEMANTICS` | 100 | Compile-time SQL semantics check |
| `METHOD_CALLBACK_GET_GLOBAL_SEMANTICS` | 101 | Compile-time global semantics check |
| `METHOD_CALLBACK_CHANGE_RIGHTS` | 200 | Switch execution rights (definer ↔ invoker) |
| `METHOD_CALLBACK_GET_CODE_ATTR` | 201 | Fetch SP code attributes |

> [!key-insight] Bidirectional callback loop
> During a single `SP_CODE_INVOKE` exchange, `cub_pl` can send multiple `SP_CODE_INTERNAL_JDBC` packets back to `cub_server`. Each callback is fully serviced (prepare/execute/fetch/…) before reading the next packet. The loop in `response_invoke_command()` continues until `SP_CODE_RESULT` or `SP_CODE_ERROR` is received. This means a single SP call holds one socket connection and one execution_stack for its entire duration, including nested SQL execution in the Java body.

## Connection Pool Epoch

`connection_pool::m_epoch` is an atomic counter. It increments whenever `cub_pl` restarts (`server_manager` calls `increment_epoch()` after re-fork). Each `connection` stores the epoch at which it was created; on `claim()`, if the connection epoch is stale, `do_reconnect()` is called transparently. This prevents callers from using sockets to a dead `cub_pl` process after automatic restart.

## Header Structure

Each message is prefixed by a `cubmethod::header` containing a `command` field set to the appropriate `SP_CODE_*` or `METHOD_CALLBACK_*` value. The `execution_stack` maintains two headers:

- `m_java_header` — used for messages to `cub_pl`
- `m_client_header` — used for callback messages to the CAS client

`set_java_command(int)` and `set_cs_command(int)` update these before packing.

## Related

- [[components/sp|sp]] — hub
- [[components/sp-jni-bridge|sp-jni-bridge]] — marshalling on top of this protocol
- [[modules/pl_engine|pl_engine]] — Java side: `ExecuteThread.java` implements the server end
