---
type: component
parent_module: "[[modules/src|src]]"
path: "src/connection/"
status: developing
purpose: "CSS (CUBRID Server/client Support) protocol layer: TCP socket management, cub_master coordination, HA heartbeat, request/response packet framing"
key_files:
  - "connection_defs.h (CSS_CONN_ENTRY, NET_HEADER, all enums)"
  - "connection_sr.c (server-side connection pool: css_Conn_array)"
  - "connection_cl.h / connection_cl.cpp (client-side connection_cl class)"
  - "connection_support.cpp (shared css_readn / css_writen utilities)"
  - "connection_support.hpp (connection_support base class)"
  - "server_support.c (server request dispatch, HA state machine)"
  - "client_support.c / client_support.h (client request sending)"
  - "tcp.c / tcp.h (raw socket open/close/accept/transfer-fd)"
  - "heartbeat.c / heartbeat.h (HA heartbeat protocol)"
  - "master_connector.cpp / master_connector.hpp (epoll-driven master link)"
  - "connection_pool.hpp (cubconn::connection::pool — server-side conn pool)"
  - "connection_worker.hpp (cubconn::connection::worker)"
  - "connection_globals.c / connection_globals.h (css_Conn_array, global state)"
public_api:
  - "css_connect_to_cubrid_server(host, server_name) -> CSS_CONN_ENTRY*"
  - "css_connect_to_master_server(port, server_name, len) -> CSS_CONN_ENTRY*"
  - "css_send_data_packet() / css_receive_data_packet()"
  - "css_send_heartbeat_request() / css_receive_heartbeat_request()"
  - "hb_register_to_master(conn, type)"
  - "css_tcp_client_open(host, port) -> SOCKET"
  - "css_tcp_master_open(port, sockfd[2])"
  - "css_transfer_fd(server_fd, client_fd, rid, request)"
tags:
  - component
  - cubrid
  - connection
  - networking
  - ha
related:
  - "[[modules/src|src]]"
  - "[[components/cub-master|cub-master]]"
  - "[[components/heartbeat|heartbeat]]"
  - "[[components/tcp-layer|tcp-layer]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[modules/broker|broker]]"
  - "[[components/thread|thread]]"
  - "[[Architecture Overview]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[sources/cubrid-src-connection|cubrid-src-connection]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/connection/` — Client-Server Connection Layer

This directory implements the entire **CSS (CUBRID Server/client Support)** protocol: from raw TCP socket primitives up through the cub_master coordination daemon and HA heartbeat. Everything a client or broker needs to open, authenticate, and exchange packets with a `cub_server` process lives here.

## Architecture Overview

```
 Client (CS_MODE / CAS)
        │
        │  css_connect_to_cubrid_server()
        │    └─ css_connect_to_master_server()   [DATA_REQUEST]
        ▼
 ┌─────────────────────────────┐
 │   cub_master (master_sr.c)  │   listens on TCP port + Unix domain socket
 │   master_connector.cpp      │   epoll-driven, sees all server registrations
 │   (cubconn::master::connector) │
 └──────────────┬──────────────┘
                │ css_transfer_fd() — passes client socket fd to server
                │  (SCM_RIGHTS sendmsg over Unix domain socket)
                ▼
 ┌─────────────────────────────┐
 │     cub_server (SERVER_MODE)│
 │   css_Conn_array[]          │  fixed-size array, one CSS_CONN_ENTRY per client
 │   connection_pool (epoll)   │  cubconn::connection::pool dispatches workers
 │   css_server_task           │  cubthread::entry_task per request
 └─────────────────────────────┘
```

### Two-Socket Master Design

`css_tcp_master_open()` opens **two** listening sockets simultaneously:
- `sockfd[0]`: `AF_INET` TCP — for remote clients and cross-host broker connections
- `sockfd[1]`: `AF_UNIX` — for local clients at `$CUBRID_TMP/<prefix><port>` — automatically preferred by `css_sockaddr()` when host resolves to `127.0.0.1`

### FD Passing

When cub_master accepts a new client and routes it to a `cub_server`, it calls `css_transfer_fd()`. This uses `sendmsg()` with `SCM_RIGHTS` to pass the open client socket file descriptor directly to the server process — no data is proxied; once connected the server owns the fd.

## Protocol Layers

| Layer | Files | Description |
|-------|-------|-------------|
| Raw TCP | `tcp.c` | `connect`, `bind`, `listen`, `accept`, `poll` |
| CSS framing | `connection_support.cpp` | `css_readn` / `css_writen` (handles short reads/writes) |
| Packet dispatch | `connection_sr.c`, `connection_cl.h` | queue COMMAND / DATA / ERROR / ABORT / CLOSE packets |
| Request dispatch | `server_support.c` | `css_internal_request_handler` → registered handler table |
| HA heartbeat | `heartbeat.c` | separate keep-alive channel to master |

See [[components/network-protocol|network-protocol]] for the `NET_HEADER` packet format.

## `CSS_CONN_ENTRY` — Per-Connection State

Defined in `connection_defs.h`. The central data structure for both client and server sides:

| Field | Server only? | Purpose |
|-------|-------------|---------|
| `fd` | No | Socket file descriptor (must be first) |
| `request_id` | No | Monotonically increasing; upper 16 bits = entry id |
| `status` | No | `CONN_OPEN / CONN_CLOSED / CONN_CLOSING` |
| `transaction_id` | No | Currently active transaction |
| `pending_request_count` | Yes (`std::atomic`) | Requests received, not yet dispatched |
| `working_task_count` | Yes (`std::atomic`) | Tasks currently executing in thread pool |
| `request_queue / data_queue` | Both | Singly-linked queues of `CSS_QUEUE_ENTRY` |
| `rmutex / cmutex` | Yes | Per-connection recursive mutexes |
| `worker` | Yes | `cubconn::connection::worker*` — epoll worker |
| `session_p` | Yes | Current session state |
| `stop_talk` | Yes | Block/drain this connection |
| `in_method` | No | Connection used for method callback |

Server connections live in `css_Conn_array[]` (global fixed-size array). Free and active lists are maintained via `css_Free_conn_anchor` / `css_Active_conn_anchor` with RW-locks.

## Connection Lifecycle (Server-Side)

```
css_tcp_master_open()
   → (master) accept client
   → css_transfer_fd() to server
   → (server) css_open_new_socket_from_master()  -- recvmsg SCM_RIGHTS
   → css_Conn_array slot allocated
   → cubconn::connection::pool::dispatch(conn)
   → cubconn::connection::worker receives epoll event
   → css_server_task::execute() dispatched to thread pool
   → css_internal_request_handler()
   → registered handler (NET_HEADER.function_code)
   → response packet sent
   → CONN_CLOSING → css_dealloc_conn()
```

## Build Mode Behavior

| Mode | Behavior |
|------|----------|
| `SERVER_MODE` | `connection_sr.c`, `server_support.c` active; `css_Conn_array` exists |
| `CS_MODE` | `connection_cl.h/cpp`, client-side; connects via master to remote server |
| `SA_MODE` | Files compiled in but no TCP sockets opened; in-process dispatch |

> [!warning] SA_MODE compilation
> `connection_cl.c`, `connection_less.c`, `connection_globals.c`, `connection_list_cl.c`, and `connection_support.cpp` are all compiled into the SA build even though no TCP connections are established. Guards inside redirect to in-process call paths.

## Sub-Components

- [[components/cub-master|cub-master]] — master process role, registration protocol, HA management
- [[components/network-protocol|network-protocol]] — `NET_HEADER` packet format, request IDs, function codes
- [[components/heartbeat|heartbeat]] — HA heartbeat: node states, failover detection, register/deregister
- [[components/tcp-layer|tcp-layer]] — raw socket primitives: open, retry, sockopt, fd passing, peer-alive

## Integration

- [[modules/broker|broker]] processes (CAS) use `css_connect_to_cubrid_server()` as their primary server entry point.
- [[components/thread|thread]] worker pools (`cubthread`) execute `css_server_task` objects dispatched by the connection pool.
- [[Build Modes (SERVER SA CS)]] governs which files are compiled and whether actual network I/O occurs.
- [[Architecture Overview]] describes the overall client → broker → CAS → DB server topology.

## Related

- Parent: [[modules/src|src]]
- Source: [[sources/cubrid-src-connection|cubrid-src-connection]]
