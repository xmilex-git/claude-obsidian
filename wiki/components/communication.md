---
type: component
parent_module: "[[modules/src|src]]"
path: "src/communication/"
status: developing
purpose: "Client/server network interface: NET_SERVER_REQUEST_LIST dispatch table, request packing/unpacking, server-side handler registration, client request sending, method/xs callback glue, per-request histogram collection"
key_files:
  - "network.h (NET_SERVER_REQUEST_LIST X-macro, ~250+ request constants, capability flags)"
  - "network_request_def.hpp (net_request struct, net_req_act flags, net_server_func typedef — SERVER_MODE only)"
  - "network_common.cpp (net_server_request_name[] string table, get_net_request_name())"
  - "network_cl.c (CS-mode client support: send requests, error mapping, capability checks)"
  - "network_interface_cl.c (client-side interface: bridges higher-level APIs to network layer)"
  - "network_sr.c (server-side: net_server_init(), net_Requests[] dispatch table)"
  - "network_interface_sr.c (server-side request processing, enter/exit server logic)"
  - "network_callback_cl.hpp/cpp (client callback glue: xs_pack_and_queue, xs_send_queue)"
  - "network_callback_sr.hpp/cpp (server callback glue: xs_callback_send/receive)"
  - "network_histogram.hpp/cpp (client per-request histogram: net_histo_ctx)"
public_api:
  - "get_net_request_name(int request) -> const char*"
  - "net_server_func: void (*)(THREAD_ENTRY*, unsigned int rid, char* request, int reqlen)"
  - "xs_pack_and_queue(Args...) -> int   [client callback template]"
  - "xs_callback_send_and_receive(thread_p, func, Args...) -> int   [server callback template]"
  - "histo_start(bool) / histo_stop() / histo_print(FILE*)"
tags:
  - component
  - cubrid
  - communication
  - network
  - rpc
  - dispatch
related:
  - "[[components/connection|connection]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[components/packer|packer]]"
  - "[[components/request-response|request-response]]"
  - "[[components/xasl-stream|xasl-stream]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[sources/cubrid-src-communication|cubrid-src-communication]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/communication/` — Client/Server Network Interface Layer

This directory is the **structured RPC layer** sitting between the raw CSS TCP socket machinery (`src/connection/`) and the actual server logic (`src/transaction/`, `src/storage/`, etc.). Its job is to map request codes to handler functions, enforce pre/post-processing rules, provide type-safe packing helpers, and instrument client-side request performance.

> **Distinction from `src/connection/`**
> `src/connection/` = raw TCP, CSS framing, cub_master, HA heartbeat, `CSS_CONN_ENTRY`.
> `src/communication/` = structured RPC on top: what request code goes to which function, how arguments are serialized, what authorization/transaction checks run before the handler.

## Architecture Overview

```
 Client (CS_MODE / SA_MODE)
        │
        │  network_interface_cl.c  ─────────────────────────────┐
        │  "send NET_SERVER_QM_QUERY_EXECUTE + packed args"      │
        ▼                                                        │
 ┌─────────────────────┐                                         │
 │  network_cl.c        │   CS_MODE only                         │
 │  · capability check  │   → css_send_data_packet()             │
 │  · histogram track   │     (src/connection css framing)       │
 └─────────┬───────────┘                                         │
           │ [TCP bytes]                                         │
           ▼                                                     │
 ┌─────────────────────┐                                         │
 │  cub_server         │                                         │
 │  server_support.c   │  css_internal_request_handler →         │
 │  → net_server_request()                                       │
 │     ├─ check action_attribute flags (net_req_act bitmask)     │
 │     │    CHECK_DB_MODIFICATION, CHECK_AUTHORIZATION,          │
 │     │    IN_TRANSACTION, OUT_TRANSACTION, SET_DIAGNOSTICS_INFO│
 │     └─ call net_Requests[code].processing_function()          │
 │           network_interface_sr.c implementations              │
 └─────────────────────────────────────────────────────────────┘
           │ response (packed bytes via css_send_data_packet)
           ▼
 network_interface_cl.c → unpack response → return to caller
```

### SA_MODE Shortcut

In SA_MODE, `network_interface_cl.c` bypasses the network entirely. After an `enter_server()` call that increments `db_on_server`, it calls the `xserver_interface.h` function directly. `exit_server()` then decrements the counter and restores error state.

## The Dispatch Table

### `NET_SERVER_REQUEST_LIST` — X-Macro Enumeration

`network.h` defines every server request via an X-macro:

```c
#define NET_SERVER_REQUEST_LIST \
  NET_SERVER_REQUEST_ITEM(NET_SERVER_PING) \
  NET_SERVER_REQUEST_ITEM(NET_SERVER_BO_INIT_SERVER) \
  NET_SERVER_REQUEST_ITEM(NET_SERVER_TM_SERVER_COMMIT) \
  /* ... ~250 total entries ... */
  NET_SERVER_REQUEST_ITEM(NET_SERVER_QM_QUERY_EXECUTE) \
  NET_SERVER_REQUEST_ITEM(NET_SERVER_LOGWR_GET_LOG_PAGES) \
  /* ... */
```

The macro is expanded three ways:
1. **Enum** — `NET_SERVER_REQUEST_ITEM(name) name,` → integer constants
2. **String table** — `NET_SERVER_REQUEST_ITEM(name) #name,` → `net_server_request_name[]` in `network_common.cpp`
3. **Histogram array** — indexed by the same integer constants

### Request Groups (by prefix)

| Prefix | Subsystem | Example |
|--------|-----------|---------|
| `BO_` | Boot / server management | `BO_REGISTER_CLIENT`, `BO_CHANGE_HA_MODE` |
| `TM_` | Transaction manager | `TM_SERVER_COMMIT`, `TM_SERVER_2PC_PREPARE` |
| `LC_` | Locator (object fetch/store) | `LC_FETCH`, `LC_FORCE` |
| `HEAP_` | Heap operations | `HEAP_CREATE`, `HEAP_HAS_INSTANCE` |
| `LOG_` | Log/LOB | `LOG_CHECKPOINT`, `LOG_FIND_LOB_LOCATOR` |
| `LK_` | Lock dump | `LK_DUMP` |
| `BTREE_` | B-tree | `BTREE_ADDINDEX`, `BTREE_FIND_UNIQUE` |
| `DISK_` | Disk/volume | `DISK_TOTALPGS`, `DISK_VLABEL` |
| `QST_` | Query statistics | `QST_UPDATE_STATISTICS` |
| `QM_` | Query manager | `QM_QUERY_PREPARE`, `QM_QUERY_EXECUTE` |
| `LS_` | List file (temp results) | `LS_GET_LIST_FILE_PAGE` |
| `MNT_` | Monitor/perf stats | `MNT_SERVER_COPY_STATS` |
| `SES_` | Session state | `SES_CHECK_SESSION`, `SES_CREATE_PREPARED_STATEMENT` |
| `PRM_` | Parameters | `PRM_SET_PARAMETERS`, `PRM_GET_PARAMETERS` |
| `REPL_` | Replication | `REPL_INFO`, `REPL_LOG_GET_APPEND_LSA` |
| `LOGWR_` | Log writer (HA standby) | `LOGWR_GET_LOG_PAGES` |
| `ES_` | External storage (LOB) | `ES_CREATE_FILE`, `ES_READ_FILE` |
| `JSP_` | Java stored procedures | `JSP_GET_SERVER_PORT` |
| `CSS_` | Connection/thread control | `CSS_KILL_TRANSACTION` |
| `TDE_` | Transparent data encryption | `TDE_GET_DATA_KEYS` |

### `net_request` Struct and `net_req_act` Flags

Defined in `network_request_def.hpp` (SERVER_MODE only, enforced by `#error`):

```cpp
enum net_req_act
{
  CHECK_DB_MODIFICATION = 0x0001,  // reject if DB is read-only
  CHECK_AUTHORIZATION   = 0x0002,  // reject if not DBA
  SET_DIAGNOSTICS_INFO  = 0x0004,  // populate diagnostics after call
  IN_TRANSACTION        = 0x0008,  // must be in an active transaction
  OUT_TRANSACTION       = 0x0010,  // ends (commits/aborts) the transaction
};

typedef void (*net_server_func) (THREAD_ENTRY *thrd, unsigned int rid, char *request, int reqlen);

struct net_request
{
  int action_attribute;               // bitmask of net_req_act
  net_server_func processing_function; // handler pointer
};
```

`net_Requests[]` is a `static struct net_request` array of size `NET_SERVER_REQUEST_END`, populated once by `net_server_init()` at server startup. The `server_support.c` dispatcher checks flags before calling `processing_function`.

### Example Entries

```c
// transaction commit — checks write, sets diagnostics, ends transaction
req_p->action_attribute = (CHECK_DB_MODIFICATION | SET_DIAGNOSTICS_INFO | OUT_TRANSACTION);
req_p->processing_function = stran_server_commit;

// query execute — sets diagnostics, must be in transaction
req_p->action_attribute = (SET_DIAGNOSTICS_INFO | IN_TRANSACTION);
req_p->processing_function = sqmgr_execute_query;

// backup — requires DBA auth + transaction
req_p->action_attribute = (CHECK_AUTHORIZATION | IN_TRANSACTION);
req_p->processing_function = sboot_backup;
```

## Method / XS Callback Glue

Method callbacks (used for Java stored procedures and XASL method invocations) use a bidirectional sub-protocol within a request:

### Client side (`network_callback_cl.hpp`)

```cpp
// Pack args and enqueue in xs_get_data_queue():
template <typename... Args>
int xs_pack_and_queue(Args&&... args);

// Pack + immediately send (CS_MODE calls xs_queue_send()):
template <typename... Args>
int xs_send_queue(Args&&... args);
```

A `std::queue<cubmem::extensible_block>` buffers outgoing callback blocks.

### Server side (`network_callback_sr.hpp`)

```cpp
// Pack args into extensible_block and send to client:
template <typename... Args>
int xs_callback_send_args(THREAD_ENTRY*, Args&&...);

// Send, then block waiting for client callback response:
template <typename... Args>
int xs_callback_send_and_receive(THREAD_ENTRY*, xs_callback_func, Args&&...);
```

Both sides rely on `packing_packer::set_buffer_and_pack_all()` (see [[components/packer]]).

## Per-Request Histogram (`network_histogram.hpp`)

CS-mode-only instrumentation. `net_histo_ctx` holds:

```cpp
std::array<net_histogram_entry, NET_SERVER_REQUEST_END> histogram_entries;
// Each entry: request_count, total_size_sent, total_size_received, elapsed_time
```

Public API (also compiled in SA_MODE as stubs):

```c
histo_start(bool for_all_trans);
histo_stop();
histo_print(FILE *stream);
histo_print_global_stats(FILE *stream, bool cumulative, const char *substr);
histo_add_request(int request, int sent);
histo_finish_request(int request, int received);
```

## Build Integration

| File | Server (`cubrid/`) | CS (`cs/`) | SA (`sa/`) |
|------|:--:|:--:|:--:|
| `network_common.cpp` | ✓ | ✓ | — |
| `network_sr.c` | ✓ | — | — |
| `network_interface_sr.c` | ✓ | — | — |
| `network_callback_sr.cpp` | ✓ | — | ✓ |
| `network_cl.c` | — | ✓ | — |
| `network_interface_cl.c` | — | ✓ | ✓ |
| `network_callback_cl.cpp` | — | ✓ | ✓ |
| `network_histogram.cpp` | — | ✓ | ✓ |

> [!warning] SA_MODE compilation
> `network_callback_sr.cpp` is compiled into SA builds to support method/stored-procedure callbacks even without a network. `network_interface_cl.c` dispatches directly to `xserver_interface.h` functions in SA_MODE — `db_on_server` counter guards re-entrancy.

## Adding New Requests

1. Add `NET_SERVER_REQUEST_ITEM(NET_SERVER_MY_FEATURE)` to the macro list in `network.h`.
2. Register a handler in `network_sr.c` inside `net_server_init()`:
   ```c
   req_p = &net_Requests[NET_SERVER_MY_FEATURE];
   req_p->action_attribute = IN_TRANSACTION;
   req_p->processing_function = smy_feature_handler;
   ```
3. Implement the server handler in `network_interface_sr.c`.
4. Add the client-side call in `network_interface_cl.c`.

## Sub-Components

- [[components/packer|packer]] — `cubpacking::packer` / `unpacker` type-safe byte-stream serialization
- [[components/request-response|request-response]] — dispatch table mechanics, `net_req_act` flags, handler registration
- [[components/connection|connection]] — lower layer: CSS framing, sockets, `CSS_CONN_ENTRY`, cub_master

## Integration

- [[components/connection|connection]] provides the raw CSS packets that arrive at `server_support.c → net_server_request()`.
- [[components/xasl-stream|xasl-stream]] uses the same `packing_packer` infrastructure for XASL serialization.
- [[components/sp-protocol|sp-protocol]] uses the callback glue (`xs_callback_send_and_receive`) for Java SP invocations.
- [[Build Modes (SERVER SA CS)]] governs which source files are compiled and whether `network_interface_cl.c` goes to network or calls server functions directly.

## Related

- Parent: [[modules/src|src]]
- Source: [[sources/cubrid-src-communication|cubrid-src-communication]]
