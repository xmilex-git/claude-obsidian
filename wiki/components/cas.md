---
type: component
parent_module: "[[modules/broker|broker]]"
path: "src/broker/"
status: developing
purpose: "CAS (CUBRID Application Server) worker process — one process per client session, handles protocol dispatch and SQL execution, bridges driver protocol to CSS/db_* API"
key_files:
  - "cas.c — entry point, main(), server_fn_table[], cas_main(), shard_cas_main()"
  - "cas_common_main.c — cas_main_loop() shared by CAS and shard CAS"
  - "cas_execute.c — SQL execution: fn_prepare, fn_execute, fn_fetch, fn_end_tran"
  - "cas_function.c — fn_* handler implementations for non-SQL protocol ops"
  - "cas_network.c — network read/write wrappers for client socket"
  - "cas_net_buf.c — T_NET_BUF output buffer: append-then-flush pattern"
  - "cas_handle.c — T_SRV_HANDLE (statement handle) and T_REQ_HANDLE lifecycle"
  - "cas_log.c / cas_sql_log2.c — per-CAS SQL logging"
  - "cas_ssl.c — optional SSL/TLS wrapping of client socket"
  - "cas_protocol.h — CAS_FC_* function codes, protocol version constants"
tags:
  - component
  - cubrid
  - cas
  - connection-pooling
  - multi-process
  - protocol-dispatch
related:
  - "[[components/broker-impl|broker-impl]]"
  - "[[components/broker-shm|broker-shm]]"
  - "[[components/connection|connection]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[components/client-api|client-api]]"
  - "[[modules/broker|broker]]"
  - "[[Architecture Overview]]"
  - "[[sources/cubrid-src-broker|cubrid-src-broker]]"
created: 2026-04-23
updated: 2026-04-23
---

# CAS — CUBRID Application Server Worker Process

CAS (CUBRID Application Server) is the **worker process** in CUBRID's three-tier architecture. One CAS process exists per concurrent client connection. Broker spawns CAS children and dispatches client sockets to them; CAS processes open their own session with `cub_server` using the standard CSS connection layer.

## Process Lifecycle

```
broker forks/execs → cub_cas binary

main() [cas.c]
  ├─ signal handlers: SIGTERM, SIGSEGV, SIGABRT, SIGFPE, SIGILL, SIGBUS, SIGSYS
  │    (SIGPIPE, SIGUSR1, SIGXFSZ → SIG_IGN)
  ├─ cas_init()
  │    ├─ cas_init_shm()    # attach T_SHM_APPL_SERVER, find own as_info[idx]
  │    └─ set_cubrid_home()
  └─ if shard_flag:  shard_cas_main()
     else:           cas_main()
                      └─ cas_main_loop(&ops)
```

### `cas_main_loop` — The Core Request Loop

`cas_main_loop()` lives in `cas_common_main.c` and is parameterised via a `CAS_MAIN_OPS` callback struct. This allows `cas.c` (standard) and the shard CAS variant to reuse one loop:

```c
typedef struct cas_main_ops {
  void (*init_specific)(void);
  void (*pre_db_connect)(void);
  int  (*db_connect)(SOCKET, const char *db, const char *user,
                     const char *passwd, const char *url,
                     T_REQ_INFO *, char *cas_info);
  void (*post_db_connect)(...);
  void (*cleanup_session)(void);
  FN_RETURN (*process_request)(SOCKET, T_NET_BUF *, T_REQ_INFO *, SOCKET);
  void (*set_session_id)(T_CAS_PROTOCOL, char *session);
  void (*send_connect_reply)(T_CAS_PROTOCOL, SOCKET, char *cas_info);
  void *context;
} CAS_MAIN_OPS;
```

Loop phases (standard CAS path):

```
1. Wait for client fd from broker (recv via broker_send_fd / Unix socket)
2. Read connection header: db_name, db_user, db_passwd, url
3. ops.db_connect()
   └─ css_connect_to_cubrid_server(db_host, db_name)  [src/connection/]
   └─ db_login() / db_restart()                        [src/compat/]
4. ops.post_db_connect()
   └─ update shm_appl->as_info[idx]: last_connect_time, client IP, driver_version
5. ops.send_connect_reply() → write session key + session ID to client
6. Request loop:
   ┌─ read MSG_HEADER (func_code, request_size) from client socket
   │  set shm_appl->as_info[idx].fn_status = func_code
   ├─ ops.process_request()
   │   └─ server_fn_table[func_code]()
   │        → fn_prepare / fn_execute / fn_fetch / fn_end_tran / ...
   └─ if con_status == CON_STATUS_CLOSE: exit loop
7. ops.cleanup_session()
   └─ db_shutdown() or db_abort_transaction()
8. Set shm uts_status = UTS_STATUS_IDLE (if keep_connection=ON)
   or exit process (if keep_connection=OFF)
```

## Protocol Function Dispatch

`server_fn_table[]` in `cas.c` is indexed by `CAS_FC_*` codes from `cas_protocol.h`. All handler signatures follow `FN_RETURN fn_xxx(T_SRV_HANDLE *, int argc, void **argv, T_NET_BUF *, T_REQ_INFO *)`.

| Category | Functions |
|----------|-----------|
| Transaction | `fn_end_tran`, `fn_savepoint`, `fn_xa_prepare`, `fn_xa_recover`, `fn_xa_end_tran` |
| Statement | `fn_prepare`, `fn_execute`, `fn_prepare_and_execute`, `fn_execute_batch`, `fn_execute_array` |
| Result | `fn_fetch`, `fn_cursor`, `fn_next_result`, `fn_cursor_update`, `fn_cursor_close` |
| Schema | `fn_schema_info`, `fn_get_attr_type_str`, `fn_parameter_info` |
| Object | `fn_oid_get`, `fn_oid_put`, `fn_oid`, `fn_collection` |
| LOB | `fn_lob_new`, `fn_lob_write`, `fn_lob_read` |
| Session | `fn_end_session`, `fn_check_cas`, `fn_con_close`, `fn_set_cas_change_mode` |
| Misc | `fn_get_db_version`, `fn_get_db_parameter`, `fn_set_db_parameter`, `fn_get_row_count`, `fn_get_last_insert_id`, `fn_get_generated_keys`, `fn_get_query_info`, `fn_make_out_rs` |
| Deprecated | `fn_deprecated` (3 GLO entries), `fn_not_supported` (shard info) |

## SQL Execution Path (`cas_execute.c`)

```
fn_execute() / fn_prepare()
  → ux_execute() / ux_prepare()          [cas_common_execute.c]
  → db_compile_statement() / db_execute() [src/compat/db_query.c]
  → network call → cub_server            [CSS protocol, src/connection/]
  → result rows → T_NET_BUF
  → cas_network: write to client socket
```

`cas_execute.c` also manages `T_SRV_HANDLE` (server-side statement handle) and maps CUBRID result types to driver wire types.

## Session and Connection State

### Shared Memory Slot (`T_APPL_SERVER_INFO`)

CAS reads and writes its shared memory slot (`shm_appl->as_info[as_index]`) to communicate with the broker:

| Field | Set by | Meaning |
|-------|--------|---------|
| `uts_status` | both | IDLE / BUSY / CON_WAIT / RESTART |
| `con_status` | CAS | OUT_TRAN / IN_TRAN / CLOSE |
| `fn_status` | CAS | Current `CAS_FC_*` code being executed |
| `claimed_alive_time` | CAS | Updated periodically; broker hang-check compares to `time(NULL)` |
| `last_access_time` | CAS | Last client activity; used for time_to_kill |
| `pid` | broker | CAS process ID |
| `cas_clt_ip` | CAS | Connected client IP |
| `driver_version` | CAS | Version string from connection header |
| `num_queries_processed` | CAS | Running counter |

### `con_status_sem` (POSIX semaphore)

The `CON_STATUS_LOCK` macro guards concurrent modification of `con_status` and `uts_status` between broker and CAS. On Linux this is a `sem_t` embedded in the shared memory struct. On Windows it is implemented with Peterson's algorithm using `con_status_lock[2]` and `con_status_lock_turn`.

## Session ID Protocol

CAS exchanges a **session key** with the driver on connection:

```c
// PROTOCOL_V3+: 8-byte server session key + 4-byte SESSION_ID
cas_make_session_for_driver(char *out):
  memcpy(out, db_get_server_session_key(), SERVER_SESSION_KEY_SIZE); // 8 bytes
  session = htonl(db_get_session_id());
  memcpy(out+8, &session, 4);
  memset(out+12, 0, DRIVER_SESSION_SIZE - 12);

// Old drivers (<V3): always send 0xFF key → new session created
```

## Error Handling

CAS error logging is **independent** of the engine `er_set()` system:

- `cas_error_log_write()` — writes `YYYY-MM-DD HH:MM:SS` timestamped entries to `<log_dir>/<broker>_<as_index>.err`
- `cas_log_write_and_end()` — SQL log entries
- Driver receives error as a 4-byte negative integer in the response header, translated from CAS internal codes

## Integration

- **Broker** spawns and monitors CAS via [[components/broker-impl|broker-impl]]
- **Shared memory** protocol detailed in [[components/broker-shm|broker-shm]]
- **DB connection** uses `css_connect_to_cubrid_server()` from [[components/connection|connection]]
- **SQL execution** calls `db_*` API from [[components/client-api|client-api]] (`src/compat/`)
- **Shard CAS** variant described in [[components/shard-broker|shard-broker]]

## Related

- Hub: [[components/broker-impl|broker-impl]]
- Source: [[sources/cubrid-src-broker|cubrid-src-broker]]
