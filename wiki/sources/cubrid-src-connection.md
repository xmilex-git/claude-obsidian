---
type: source
title: "CUBRID src/connection/ — Client-Server TCP / Heartbeat Layer"
source_path: "src/connection/"
date_ingested: 2026-04-23
status: complete
tags:
  - cubrid
  - source
  - connection
  - networking
  - ha
related:
  - "[[components/connection|connection]]"
  - "[[components/cub-master|cub-master]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[components/heartbeat|heartbeat]]"
  - "[[components/tcp-layer|tcp-layer]]"
---

# Source: `src/connection/`

CUBRID client-server connection layer. Ingested 2026-04-23.

## Files Read

| File | Role |
|------|------|
| `AGENTS.md` | Directory manifest — CSS protocol, file roles, conventions, gotchas |
| `connection_defs.h` | All enums and data structures: `NET_HEADER`, `CSS_CONN_ENTRY`, `css_command_type`, `css_packet_type`, `ha_mode`, `ha_server_mode`, `HBP_*` |
| `tcp.c` | Raw socket layer: open, listen, accept, retry, sockopt, fd passing, peer-alive |
| `connection_sr.c` | Server-side connection pool: `css_Conn_array`, queue management, packet dispatch |
| `connection_cl.h` | Client-side `connection_cl` class: `css_connect_to_cubrid_server`, `css_connect_to_master_server` |
| `server_support.c` | Server request dispatch, HA state machine, worker pool (`css_server_task`) |
| `heartbeat.c / heartbeat.h` | HA heartbeat send/recv, process registration, CS_MODE reader thread, HBP format |
| `master_connector.cpp` | `cubconn::master::connector` — epoll-driven server-side link to cub_master |
| `connection_pool.hpp` | `cubconn::connection::pool` — epoll-based connection pool |
| `connection_support.cpp` | `css_readn` / `css_writen` — partial I/O handling |

## Pages Created

- [[components/connection|connection]] — hub page: architecture, CSS_CONN_ENTRY, build modes, lifecycle
- [[components/cub-master|cub-master]] — master process role, dual sockets, FD passing, HA management commands
- [[components/network-protocol|network-protocol]] — NET_HEADER format, packet types, request ID encoding, error codes
- [[components/heartbeat|heartbeat]] — HA heartbeat: node states, timing, HBP packet format, failover logic
- [[components/tcp-layer|tcp-layer]] — socket primitives, retry logic, sockopt, FD passing implementation

## Key Findings

1. **Dual-socket master**: `css_tcp_master_open()` creates both an `AF_INET` TCP socket and `AF_UNIX` socket simultaneously. Local connections transparently use Unix domain, eliminating TCP stack overhead.

2. **Zero-copy fd hand-off**: After initial handshake, cub_master passes the open client socket to `cub_server` via `SCM_RIGHTS` sendmsg. The master becomes invisible to further client-server communication.

3. **Failover timing**: 5 missed × 500 ms heartbeats + 3 s failover wait = ~5.5 s worst-case to elect a new primary.

4. **Partial I/O**: `css_send_data()` / `css_receive_data()` can short-write/read; `css_readn()` / `css_writen()` loop to completion.

5. **SA_MODE compilation**: Several connection files compile in SA builds with no TCP sockets opened — in-process dispatch paths instead.

6. **Heartbeat bypasses NET_HEADER**: The heartbeat channel uses raw `send()`/`recv()` without the CSS framing header — intentionally a side channel.

7. **epoll integration**: `master_connector.cpp` uses Linux epoll + eventfd for the server's persistent link back to cub_master (`cubconn::master::connector`).
