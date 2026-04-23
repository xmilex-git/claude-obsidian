---
type: component
parent_module: "[[modules/src|src]]"
path: "src/connection/"
status: developing
purpose: "Raw TCP/Unix socket primitives: open, listen, accept, retry logic, socket options, FD passing via SCM_RIGHTS, peer-alive probing"
key_files:
  - "tcp.c / tcp.h (all socket operations)"
  - "connection_support.cpp (css_readn / css_writen partial I/O loops)"
tags:
  - component
  - cubrid
  - tcp
  - networking
  - socket
related:
  - "[[components/connection|connection]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[components/cub-master|cub-master]]"
  - "[[Error Handling Convention]]"
  - "[[sources/cubrid-src-connection|cubrid-src-connection]]"
created: 2026-04-23
updated: 2026-04-23
---

# TCP Layer — Raw Socket Primitives

`tcp.c` provides all the OS-level socket operations that the CSS layer builds on. It handles both `AF_INET` (TCP) and `AF_UNIX` (Unix domain socket) connections with a transparent fallback for local connections.

## Localhost Optimization — Automatic Unix Domain Socket

`css_sockaddr(host, port, saddr, slen)` resolves the host name and builds a `sockaddr`. Crucially:

> [!key-insight] Transparent local socket upgrade
> If the resolved IP address is `127.0.0.1`, `css_sockaddr()` silently switches to `AF_UNIX` and returns the Unix domain socket path (`$CUBRID_TMP/<prefix><port>`). The caller receives a `sockaddr_un` without knowing the switch happened. This eliminates the TCP stack entirely for local client-server communication.

## Key Functions

### Client-Side

| Function | Description |
|----------|-------------|
| `css_tcp_client_open(host, port)` | Open a connection; calls `css_tcp_client_open_with_retry()` |
| `css_tcp_client_open_with_retry(host, port, will_retry)` | Blocking connect with exponential back-off (1→2→4→…→30 s sleep) |
| `css_tcp_client_open_with_timeout(host, port, timeout_ms)` | Non-blocking connect via `poll()` with millisecond timeout (Linux only) |

### Server / Master Side

| Function | Description |
|----------|-------------|
| `css_tcp_master_open(port, sockfd[2])` | Bind and listen on both TCP (`sockfd[0]`) and Unix (`sockfd[1]`) |
| `css_master_accept(sockfd)` | Blocking `accept()` loop; handles `EINTR`, logs `EMFILE`/`ENFILE` |
| `css_tcp_setup_server_datagram(path, sockfd)` | Create the server's Unix domain socket for receiving fds from master |
| `css_tcp_listen_server_datagram(sockfd, newfd)` | `accept()` on the server datagram socket |
| `css_tcp_master_datagram(path, sockfd)` | Master connects to the server's datagram socket |

### FD Passing

| Function | Description |
|----------|-------------|
| `css_transfer_fd(server_fd, client_fd, rid, request)` | Master sends `client_fd` to server via `sendmsg()` + `SCM_RIGHTS` |
| `css_open_new_socket_from_master(fd, rid)` | Server receives new fd via `recvmsg()` + `SCM_RIGHTS` |

### Utilities

| Function | Description |
|----------|-------------|
| `css_shutdown_socket(fd)` | `close()` with `EINTR` retry loop |
| `css_peer_alive(sd, timeout_ms)` | Liveness probe via connect-to-port-7 with `poll()` |
| `css_gethostname(name, len)` | Portable hostname lookup |
| `css_hostname_to_ip(host, ip_addr)` | Resolve hostname to 4-byte IP (handles `gethostbyname_r` variants) |
| `css_gethostid()` | 32-bit host identifier via `gethostid()` |
| `css_get_max_socket_fds()` | `sysconf(_SC_OPEN_MAX)` |

## Socket Options (`css_sockopt`)

Applied to every socket via `css_sockopt(sd)`:

| Option | Parameter | Effect |
|--------|-----------|--------|
| `SO_RCVBUF` | `PRM_ID_TCP_RCVBUF_SIZE` | Kernel receive buffer size |
| `SO_SNDBUF` | `PRM_ID_TCP_SNDBUF_SIZE` | Kernel send buffer size |
| `TCP_NODELAY` | `PRM_ID_TCP_NODELAY` | Disable Nagle algorithm |
| `SO_KEEPALIVE` | `PRM_ID_TCP_KEEPALIVE` | Enable TCP keep-alive probes |

All options are conditional on their system parameter being non-zero/true. `TCP_NODELAY` is also forced on newly accepted sockets in `css_master_accept()` for `AF_INET` connections.

## Retry Logic in `css_tcp_client_open_with_retry`

```
start_contime = time(NULL)
sleep_nsecs = 1
do {
  sd = socket(...)
  css_sockopt(sd)
  connect(sd, ...)
  on ECONNREFUSED or ETIMEDOUT:
    nsecs = elapsed - PRM_ID_TCP_CONNECTION_TIMEOUT
    if nsecs >= 0 AND retries > TCP_MIN_NUM_RETRIES (3):
      give up
    else:
      sleep(min(sleep_nsecs, 30, -nsecs))
      sleep_nsecs *= 2         // exponential back-off
      close(sd)
      retry
} while (success < 0 && will_retry)
```

The sleep cap of 30 seconds prevents excessive delays. The minimum of 3 retries (`TCP_MIN_NUM_RETRIES`) guarantees at least a few attempts regardless of the timeout.

## Error Handling

All socket errors follow the [[Error Handling Convention]]:
- `er_set_with_oserror(ER_ERROR_SEVERITY, ARG_FILE_LINE, ERR_CSS_TCP_*, ...)` for socket errors
- Functions return `INVALID_SOCKET` or negative error codes on failure
- `EINTR` is always retried with a `goto` label (`again_eintr:`)
- `css_shutdown_socket()` is always used instead of bare `close()` to handle `EINTR`

## FD Passing Details

`css_transfer_fd()` uses the POSIX `SCM_RIGHTS` mechanism:

1. Sends a 4-byte request code to the server's control channel (`send()`).
2. Constructs a `msghdr` with `iov` carrying the 2-byte request_id (network byte order).
3. Attaches a `cmsghdr` with `cmsg_level=SOL_SOCKET`, `cmsg_type=SCM_RIGHTS`, carrying the `client_fd` integer.
4. Calls `sendmsg()` — the OS duplicates the fd into the server process's fd table.

`css_open_new_socket_from_master()` mirrors this with `recvmsg()`, extracts the fd from the `SCM_RIGHTS` control message, and calls `fcntl(F_SETOWN)` + `css_sockopt()` before returning.

> [!warning] Linux vs other UNIX
> `css_transfer_fd()` uses `msg_accrights` on non-Linux/non-AIX systems and the `cmptr` + `CMSG_DATA` style on Linux/AIX. The `cmptr` static pointer is malloc'd once and reused — not thread-safe if multiple threads in master call `css_transfer_fd()` concurrently.

## Thread Safety Notes

- `gethostbyname` is not thread-safe on all platforms. On systems without `gethostbyname_r`, a `pthread_mutex_t gethostbyname_lock` serialises all hostname lookups.
- `css_get_master_domain_path()` is lazily initialised with a `static bool need_init` guard — not protected by a mutex, acceptable because the value is deterministic and writes are idempotent.

## Related

- [[components/connection|connection]] — full layer overview
- [[components/network-protocol|network-protocol]] — packet framing above this layer
- [[components/cub-master|cub-master]] — uses `css_tcp_master_open`, `css_transfer_fd`
