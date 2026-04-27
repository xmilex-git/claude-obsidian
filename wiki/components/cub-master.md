---
type: component
parent_module: "[[modules/src|src]]"
path: "src/connection/ + src/executables/master_*.c"
status: developing
purpose: "cub_master — the per-host coordinator daemon that brokers client-to-server connections and manages HA process registration"
key_files:
  - "master_connector.cpp / master_connector.hpp (server-side epoll link to master)"
  - "connection_defs.h (css_command_type, css_client_request, css_master_response enums)"
  - "server_support.c (css_Master_conn, css_get_master_request, HA state transitions)"
  - "tcp.c (css_tcp_master_open, css_master_accept, css_transfer_fd)"
  - "heartbeat.c (hb_register_to_master, hb_deregister_from_master)"
tags:
  - component
  - cubrid
  - master
  - ha
  - networking
related:
  - "[[components/connection|connection]]"
  - "[[components/heartbeat|heartbeat]]"
  - "[[components/tcp-layer|tcp-layer]]"
  - "[[Architecture Overview]]"
  - "[[sources/cubrid-src-connection|cubrid-src-connection]]"
created: 2026-04-23
updated: 2026-04-23
---

# cub_master — Master Process

`cub_master` is the **per-host coordinator daemon** for CUBRID. It runs as a single long-lived process per machine, listens on the configured TCP port (default varies by `PRM_ID_MASTER_PORT_ID`) plus a Unix domain socket, and serves three distinct roles:

1. **Connection routing** — accepts incoming client connections and passes the socket file descriptor to the appropriate `cub_server` via `css_transfer_fd()`.
2. **Server registration** — each `cub_server` instance announces itself to cub_master at startup and receives a persistent control channel back.
3. **HA process supervision** — tracks `copylogdb` and `applylogdb` HA helper processes; relays HA mode changes; handles heartbeat signaling.

## Dual Listen Sockets

`css_tcp_master_open(port, sockfd[2])` creates two listening sockets:

| Index | Family | Use |
|-------|--------|-----|
| `sockfd[0]` | `AF_INET` | Remote TCP connections |
| `sockfd[1]` | `AF_UNIX` | Local connections via `$CUBRID_TMP/<prefix><port>` |

When a client resolves the server host to `127.0.0.1`, `css_sockaddr()` transparently switches to the Unix domain path — avoiding the TCP stack for local traffic.

## Connection Command Types

Defined in `connection_defs.h` as `enum css_command_type`:

| Command | Value | Sender | Meaning |
|---------|-------|--------|---------|
| `INFO_REQUEST` | 1 | Admin client | Query runtime state from master |
| `DATA_REQUEST` | 2 | DB client / CAS | Open connection to a named server |
| `SERVER_REQUEST_FROM_SERVER` | 3 | cub_server | Register new server instance |
| `SERVER_REQUEST_FROM_CLIENT` | 4 | Client | Attach existing client to server |
| `SERVER_REQUEST_NEW` | 5 | cub_server | New-style server registration |

## Master Response Codes

| Response | Meaning |
|----------|---------|
| `SERVER_ALREADY_EXISTS` | Another server with same name is running |
| `SERVER_REQUEST_ACCEPTED` | Legacy acceptance |
| `SERVER_REQUEST_ACCEPTED_NEW` | New-style acceptance |
| `DRIVER_NOT_FOUND` | No matching server registered |

## HA Management Commands (css_client_request)

The master handles a rich set of HA management requests from admin tools and heartbeat daemons, including:

| Key Commands | Purpose |
|-------------|---------|
| `GET_HA_NODE_LIST` / `GET_HA_PROCESS_LIST` | Cluster node and process enumeration |
| `ACTIVATE_HEARTBEAT` / `DEACTIVATE_HEARTBEAT` | Enable/disable HA monitoring |
| `RECONFIG_HEARTBEAT` | Reload HA configuration at runtime |
| `DEREGISTER_HA_PROCESS_BY_PID` | Remove a dead HA helper process |
| `GET_SERVER_HA_MODE` | Query current HA role of a server |
| `DEACT_STOP_ALL` / `DEACT_CONFIRM_NO_SERVER` | Safe deactivation handshake |
| `GET_SERVER_STATE` | Used by broker to check server health |
| `START_HA_UTIL_PROCESS` | Launch a new HA utility (copylogdb/applylogdb) |

## FD Passing Protocol

After cub_master accepts a client and locates the target `cub_server`:

1. Master sends `CSS_SERVER_REQUEST` command integer to the server's control channel (`send()`).
2. Master calls `css_transfer_fd(server_fd, client_fd, rid, request)` — sends the client file descriptor via `sendmsg()` with `SCM_RIGHTS` ancillary data and a `request_id` in the iov.
3. Server calls `css_open_new_socket_from_master(fd, &rid)` — receives fd via `recvmsg()`, sets `F_SETOWN`, applies `css_sockopt()`, returns the new socket.

> [!key-insight] Zero-copy fd hand-off
> The actual client connection bytes never flow through cub_master after the initial handshake. The OS moves the open file description directly from master's process to the server process via `SCM_RIGHTS`. Master becomes invisible to further client-server communication.

## Server-Side Master Link (master_connector.cpp)

On the `cub_server` side, `cubconn::master::connector` (namespace `cubconn::master`) manages the persistent control channel back to cub_master. It uses Linux `epoll` and an `eventfd` to integrate with the server's event loop. States tracked in `master_state` enum. Registered as `REGISTER_CONNECTION(master_connector, 0)` — uses `TT_MASTER` thread entry rather than claiming a separate worker pool slot.

## HA State Transitions on Server

`server_support.c` maintains `ha_Server_state` (type `HA_SERVER_STATE`). The function `css_transit_ha_server_state()` enforces the valid state machine. `ha_Log_applier_state[]` (up to 5 entries) tracks the state of connected `applylogdb` processes — checked before allowing the server to report as fully `WORKING`.

## From the Manual (admin/control.rst, config.rst — added 2026-04-27)

> [!gap] Documented operator behaviors
> - **Auto-restart of cub_server (NEW 11.4)**: when `auto_restart_server=yes` (cubrid.conf), cub_master auto-restarts cub_server after abnormal termination (OOM kill, segfault). Disabled if a second crash occurs **within 120 s** of restart, or if start-failures exceed retry threshold. Does NOT auto-restart on normal `cubrid server stop`. Linux only.
> - **HA split-brain detection**: cub_master detects via `ha_ping_hosts` and writes `[Diagnosis]/[Success]/[Canceled]` prefixed events to `<host>.cub_master.err`.
> - **In HA mode, `cubrid server start` BYPASSES heartbeat orchestration** — operators must use `cubrid heartbeat start` instead. Documented in admin/control.rst:508-540.
> - **PL server (cub_pl) auto-managed**: starts/stops with cub_server when `stored_procedure=yes` (default since 11.4). Manual `cubrid pl start/stop` is rare. admin/control.rst:3185-3216.

See [[sources/cubrid-manual-admin]] for the full operator manual context.

## Related

- [[components/connection|connection]] — full connection layer hub
- [[components/heartbeat|heartbeat]] — heartbeat protocol details
- [[components/tcp-layer|tcp-layer]] — `css_tcp_master_open`, `css_transfer_fd` implementation
- [[Architecture Overview]] — where cub_master fits in the overall topology
- [[sources/cubrid-manual-admin]] — admin guide
- [[sources/cubrid-manual-ha]] — HA reference
