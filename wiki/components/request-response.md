---
type: component
parent_module: "[[modules/src|src]]"
path: "src/communication/"
status: developing
purpose: "Server-side request handler registration model: net_Requests[] dispatch array, net_req_act pre/post-processing flags, net_server_func handler contract, and the flow from CSS packet arrival to handler call and response send."
key_files:
  - "network.h (NET_SERVER_REQUEST_LIST X-macro, NET_SERVER_* enum)"
  - "network_request_def.hpp (net_request struct, net_req_act bitmask, net_server_func typedef)"
  - "network_sr.c (net_server_init() — populates net_Requests[])"
  - "network_interface_sr.c (implementations of all s* handler functions)"
  - "src/connection/server_support.c (dispatcher: css_internal_request_handler → net_server_request)"
tags:
  - component
  - cubrid
  - request-response
  - dispatch
  - network
  - rpc
related:
  - "[[components/communication|communication]]"
  - "[[components/connection|connection]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[components/packer|packer]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[sources/cubrid-src-communication|cubrid-src-communication]]"
created: 2026-04-23
updated: 2026-04-23
---

# Request/Response Dispatch Model

This page describes how an incoming CSS packet becomes a server function call: the registration table, the flag-driven pre/post-processing contract, and the handler function signature.

## Request Lifecycle

```
TCP bytes arrive at CSS layer (src/connection/)
  → css_internal_request_handler()   [server_support.c]
    → extract function_code from NET_HEADER
    → net_server_request(thread_p, rid, request_code, size, buffer)
      ├─ look up net_Requests[request_code]
      ├─ check action_attribute flags:
      │   CHECK_DB_MODIFICATION → reject if server is read-only
      │   CHECK_AUTHORIZATION   → reject if client is not DBA
      │   IN_TRANSACTION        → reject if no active transaction
      │   SET_DIAGNOSTICS_INFO  → set up diagnostics context (post-call)
      │   OUT_TRANSACTION       → handle transaction end after call
      └─ call processing_function(thread_p, rid, buffer, size)
           ↓
      handler unpacks arguments from buffer (packing_unpacker)
      handler calls internal engine functions
      handler packs response into reply buffer (packing_packer)
      handler calls css_send_data_packet() / net_server_send_reply()
           ↓
CSS layer delivers reply bytes to client
```

## `net_request` Structure

Defined in `network_request_def.hpp` (SERVER_MODE only — guarded by `#error`):

```cpp
typedef void (*net_server_func) (THREAD_ENTRY *thrd, unsigned int rid,
                                  char *request, int reqlen);

struct net_request
{
  int action_attribute;                // bitmask of net_req_act flags
  net_server_func processing_function; // pointer to handler function
};
```

`net_Requests[]` is a `static struct net_request[NET_SERVER_REQUEST_END]` allocated once in `network_sr.c`. It is populated during `net_server_init()` called at server startup.

## `net_req_act` Flags

```cpp
enum net_req_act
{
  CHECK_DB_MODIFICATION = 0x0001,
  CHECK_AUTHORIZATION   = 0x0002,
  SET_DIAGNOSTICS_INFO  = 0x0004,
  IN_TRANSACTION        = 0x0008,
  OUT_TRANSACTION       = 0x0010,
};
```

Flags combine with bitwise OR. The dispatcher in `server_support.c` evaluates all applicable flags before calling the handler. The handler itself does not check these conditions.

| Flag | Pre-call effect | Who sets it |
|------|----------------|-------------|
| `CHECK_DB_MODIFICATION` | Rejects request if server is in read-only mode | DDL/DML requests |
| `CHECK_AUTHORIZATION` | Rejects if client does not have DBA privilege | Admin ops: backup, stats, kill tran |
| `SET_DIAGNOSTICS_INFO` | Populates SQL diagnostics context after the call | Commit, abort, query execute |
| `IN_TRANSACTION` | Rejects if no active transaction (client not registered) | Most data access requests |
| `OUT_TRANSACTION` | Post-call transaction bookkeeping (commit/abort cleanup) | Commit, abort |

## Handler Naming Conventions

Server-side handlers in `network_interface_sr.c` follow module prefixes:

| Prefix | Module | Example |
|--------|--------|---------|
| `sboot_*` | Boot / server lifecycle | `sboot_initialize_server`, `sboot_backup` |
| `stran_*` | Transaction manager | `stran_server_commit`, `stran_server_abort` |
| `slocator_*` | Locator (object fetch/store) | `slocator_fetch`, `slocator_force` |
| `sqmgr_*` | Query manager | `sqmgr_prepare_query`, `sqmgr_execute_query` |
| `sqst_*` | Query statistics | `sqst_update_statistics` |
| `smnt_*` | Monitor/perf | `smnt_server_copy_stats` |
| `ssession_*` | Session state | `ssession_find_or_create_session` |
| `srepl_*` | Replication | `srepl_set_info`, `srepl_log_get_append_lsa` |
| `slogwr_*` | Log writer (HA standby) | `slogwr_get_log_pages` |
| `sprm_*` | System parameters | `sprm_server_change_parameters` |
| `ses_posix_*` | External storage | `ses_posix_create_file` |
| `spl_*` / `sthread_*` | JSP / thread control | `spl_get_server_port`, `sthread_kill_tran_index` |
| `svacuum` | Vacuum GC | `svacuum` |
| `server_ping` | PING | `server_ping` |

## Handler Function Signature

All handlers share the same type:

```c
void handler_function (THREAD_ENTRY *thread_p, unsigned int rid,
                       char *request, int reqlen);
```

- `thread_p` — current thread's `THREAD_ENTRY*` for memory allocation and logging
- `rid` — request ID (upper 16 bits = connection entry ID, lower 16 = monotonic counter); used when sending the reply to route it back to the correct client request
- `request` — raw byte buffer containing packed arguments
- `reqlen` — byte length of the buffer

A handler typically:
1. Creates a `packing_unpacker` over `(request, reqlen)` and unpacks arguments.
2. Calls the internal engine function (e.g., `qmgr_prepare_query()`).
3. Packs the return value and/or out-parameters into a reply buffer.
4. Calls `css_send_data_packet(conn, rid, reply_buffer, reply_size)` (or equivalent).
5. Returns `void`; errors are reported via `er_set()` and propagated through the reply.

## `NET_SERVER_REQUEST_LIST` — Stable Ordering Requirement

> [!warning] Ordering is wire-format
> Request codes are integer constants assigned by position in the `NET_SERVER_REQUEST_LIST` macro. Adding a new request in the **middle** of the list shifts all codes after it, breaking binary compatibility between client and server versions. New requests should be appended at the end (there is a comment in `network.h` acknowledging this).

The string table `net_server_request_name[]` in `network_common.cpp` is generated from the same macro, so the index always matches the enum value.

## Adding a New Request (Checklist)

1. Append `NET_SERVER_REQUEST_ITEM(NET_SERVER_MY_OP)` to the **end** of `NET_SERVER_REQUEST_LIST` in `network.h`.
2. In `network_sr.c` inside `net_server_init()`:
   ```c
   req_p = &net_Requests[NET_SERVER_MY_OP];
   req_p->action_attribute = (IN_TRANSACTION | CHECK_DB_MODIFICATION);
   req_p->processing_function = smy_op;
   ```
3. Implement `smy_op(THREAD_ENTRY*, unsigned int, char*, int)` in `network_interface_sr.c`.
4. Add the client-side sender in `network_interface_cl.c` (CS_MODE path + SA_MODE direct-call path).

## Integration

- [[components/connection|connection]] — `server_support.c` (from `src/connection/`) calls `net_server_request()` after extracting the function code from `NET_HEADER`.
- [[components/network-protocol|network-protocol]] — `NET_HEADER` carries the `function_code` that indexes into `net_Requests[]`.
- [[components/packer|packer]] — handlers use `packing_unpacker` to read arguments and `packing_packer` to write replies.
- [[Build Modes (SERVER SA CS)]] — `network_request_def.hpp` enforces `SERVER_MODE` with `#error`; SA_MODE bypasses the table entirely via direct function calls in `network_interface_cl.c`.

## Related

- Parent: [[components/communication|communication]]
- Source: [[sources/cubrid-src-communication|cubrid-src-communication]]
