---
type: component
parent_module: "[[modules/src|src]]"
path: "src/executables/master.c + src/executables/master_heartbeat.c"
status: developing
purpose: "cub_master entry point and main select() event loop — accepts connections, dispatches to registered servers, manages HA heartbeat"
key_files:
  - "master.c (main(), css_master_loop(), css_process_new_connection(), SOCKET_QUEUE_ENTRY management)"
  - "master_heartbeat.c (HA heartbeat thread inside the master process)"
  - "master_server_monitor.hpp (server_monitor — auto-restart of crashed cub_server)"
public_api:
  - "main(argc, argv)"
  - "css_master_loop() — blocking select() event loop (internal)"
  - "css_master_cleanup(sig) — exported for SIGINT handler"
tags:
  - component
  - cubrid
  - cub-master
  - ha
  - networking
  - executables
related:
  - "[[components/executables|executables]]"
  - "[[components/cub-master|cub-master]]"
  - "[[components/connection|connection]]"
  - "[[components/heartbeat|heartbeat]]"
  - "[[components/tcp-layer|tcp-layer]]"
  - "[[sources/cubrid-src-executables|cubrid-src-executables]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cub_master` — Master Process Entry Point (`master.c`)

`master.c` contains the `main()` and the entire event loop for `cub_master`. This is the per-host coordinator that routes new client connections to the correct `cub_server` process via file-descriptor passing. The protocol-level design (CSS commands, FD passing, HA registration) is documented in [[components/cub-master|cub-master]]; this page focuses on the process-level startup and the `select()` loop in `master.c`.

> [!note] Division of responsibility
> [[components/cub-master|cub-master]] documents the protocol and structure (from `src/connection/`). This page documents the `main()` in `src/executables/master.c` — the actual entry point of the `cub_master` binary.

## `main()` Flow

```
main(argc, argv)
  │
  ├─ utility_initialize() — er_init + msgcat_init
  ├─ master_util_config_startup(argv[1]) → port_id from cubrid.conf
  ├─ er_init(hostname_master.err, ER_NEVER_EXIT)
  │
  ├─ css_does_master_exist(port_id) → EXIT if already running
  │
  ├─ msgcat_final() / er_final()  ← close before daemonizing
  ├─ css_daemon_start()  ← fork() + setsid() to daemonize [Linux/Mac, unless NO_DAEMON env set]
  │
  ├─ utility_initialize() / er_init()  ← reopen after fork
  │
  ├─ css_master_init(port_id, css_Master_socket_fd[2])
  │     └─ css_tcp_master_open() → sockfd[0] = AF_INET, sockfd[1] = AF_UNIX
  │
  ├─ hb_master_init()  ← HA heartbeat (if HA not disabled)
  │
  ├─ server_monitor::reset()  ← auto-restart monitor (if auto_restart_server = true)
  │
  ├─ css_add_request_to_socket_queue() for both listening sockets
  │
  └─ css_master_loop()  ← blocks until shutdown
        css_master_cleanup(SIGINT)
```

## `css_master_loop()` — The select() Event Loop

The master runs a classic `select()`-based single-threaded event loop (timeout 4.0005 seconds):

```
css_master_loop()
  while run_code:
    select(max_fd+1, read_fd, write_fd, exception_fd, timeout=4.0005s)
    ├─ timeout (rc==0)   → css_master_timeout()
    │     checks kill(pid, 0) for each SOCKET_QUEUE_ENTRY; removes dead servers
    ├─ select error (rc==-1) → css_master_select_error()
    │     walks anchor, removes entries with invalid fds
    └─ ready fds (rc>0)  → css_check_master_socket_input()
                           css_check_master_socket_output()  (no-op)
                           css_check_master_socket_exception()
```

**`css_check_master_socket_input()`** handles two cases:
- fd == `css_Master_socket_fd[0]` or `[1]` (listening sockets) → `css_master_accept()` → `css_process_new_connection(fd)`
- other fd (existing connection): `info_p` → `css_process_info_request()`; `ha_mode` → `css_process_heartbeat_request()`; otherwise server gone → `css_remove_entry_by_conn()` (+ optionally `server_monitor` job to revive it)

## Connection Dispatch (`css_process_new_connection`)

Every new `accept()`-ed fd is classified by the first `NET_HEADER` packet received:

| `function_code` | Meaning | Handler |
|----------------|---------|---------|
| `INFO_REQUEST` | Admin info query (commdb) | `css_add_request_to_socket_queue(info_p=true)` |
| `DATA_REQUEST` | Client wants a server | `css_send_to_existing_server()` → `css_transfer_fd()` |
| `SERVER_REQUEST_FROM_SERVER` | New `cub_server` registering (port managed by master) | `css_register_new_server()` |
| `SERVER_REQUEST_FROM_CLIENT` | New server using client-style registration | `css_register_new_server(is_client=true)` |
| `SERVER_REQUEST_NEW` | New server managing its own port | `css_register_new_server2()` |

`css_transfer_fd()` uses `sendmsg()` / `SCM_RIGHTS` to pass the client socket directly to the target `cub_server` without proxying data.

## Socket Queue (`SOCKET_QUEUE_ENTRY`)

The master maintains a linked list `css_Master_socket_anchor` (mutex-protected: `css_Master_socket_anchor_lock`) of `SOCKET_QUEUE_ENTRY` nodes, one per open connection (listening sockets, server connections, info clients, HA connections). Key fields:

| Field | Purpose |
|-------|---------|
| `fd` | Open socket file descriptor |
| `pid` | PID of the registered `cub_server` (for liveness checks via `kill(pid, 0)`) |
| `port_id` | Server's own listening port (clients reconnect directly) |
| `version_string` | Server version string (sent at registration) |
| `env_var` | Server's `$CUBRID_DATABASES` env value |
| `ha_mode` | True if this is an HA-mode server |
| `info_p` | True if this is an info-only (commdb) client |
| `name` | Database name string |

## Auto-Restart (`server_monitor`)

Introduced in later versions, `master_Server_monitor` (a `std::unique_ptr<server_monitor>`) is initialized if `auto_restart_server = true` in `cubrid.conf`. When `css_check_master_socket_input()` detects a dead server, it calls:

```cpp
master_Server_monitor->produce_job(server_monitor::job_type::REVIVE_SERVER, -1, "", "", temp->name);
```

This re-spawns `cub_server <db_name>` using the stored `exec_path` and `argv` (set via `hb_set_exec_path`/`css_set_exec_path` in `server.c`).

## HA Heartbeat Integration

`master_heartbeat.c` runs HA heartbeat logic inside the `cub_master` process. Initialized by `hb_master_init()`. HA-registered servers (with `ha_mode=true` in their `SOCKET_QUEUE_ENTRY`) send heartbeat requests that are dispatched to `css_process_heartbeat_request()` in the main select loop.

See [[components/heartbeat|heartbeat]] for the heartbeat protocol and [[components/cub-master|cub-master]] for the CSS-level registration protocol.

## Daemonization

On Linux/Mac (unless `NO_DAEMON` env var is set), `css_daemon_start()` calls `fork()` + `setsid()`. The parent exits, the child continues as a daemon. Message catalog and error manager are closed before the fork and reopened after (a POSIX requirement for safety across `fork()`).

## Related

- [[components/cub-master|cub-master]] — protocol-level documentation (CSS commands, FD passing, HA registration)
- [[components/executables|executables]] — hub for all binaries
- [[components/heartbeat|heartbeat]] — HA heartbeat protocol details
- [[components/tcp-layer|tcp-layer]] — `css_tcp_master_open()` / `css_master_accept()` / `css_transfer_fd()`
