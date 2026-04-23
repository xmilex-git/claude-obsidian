---
type: component
parent_module: "[[modules/src|src]]"
path: "src/connection/"
status: developing
purpose: "CSS packet framing: NET_HEADER format, request/response model, packet types, error codes, request ID encoding"
key_files:
  - "connection_defs.h (NET_HEADER, CSS_QUEUE_ENTRY, css_packet_type, css_error_code)"
  - "connection_support.cpp (css_readn / css_writen — partial I/O handling)"
  - "connection_sr.c (server-side queue and packet dispatch)"
  - "connection_cl.h / cpp (client-side css_read_header, css_return_queued_data)"
  - "network.h (function codes — maps NET_HEADER.function_code to handler)"
tags:
  - component
  - cubrid
  - protocol
  - networking
related:
  - "[[components/connection|connection]]"
  - "[[components/cub-master|cub-master]]"
  - "[[components/tcp-layer|tcp-layer]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[sources/cubrid-src-connection|cubrid-src-connection]]"
created: 2026-04-23
updated: 2026-04-23
---

# CSS Network Protocol — Packet Format

The **CSS (CUBRID Server/client Support)** protocol is a simple request/response binary protocol layered directly on top of TCP (or Unix domain sockets for local connections). It is not HTTP and has no framing delimiter — it relies on fixed-size headers followed by variable-length data.

## NET_HEADER — Command Packet Header

Defined in `connection_defs.h` as `struct packet_header` / `typedef NET_HEADER`:

```c
struct packet_header
{
  int type;            // css_packet_type enum
  int version;         // protocol version
  int host_id;         // sender host identifier (css_gethostid())
  int transaction_id;  // active transaction index
  int request_id;      // monotonic request counter for this connection
  int db_error;        // error code from previous operation
  short function_code; // which server handler to invoke (NET_SERVER_* enum)
  unsigned short flags;// NET_HEADER_FLAG_METHOD_MODE | NET_HEADER_FLAG_INVALIDATE_SNAPSHOT
  int buffer_size;     // byte length of data payload that follows
};
```

All integer fields are sent in network byte order (big-endian). The header is always `sizeof(NET_HEADER)` bytes; the variable-length data payload immediately follows.

> [!key-insight] Partial read/write handling
> `css_send_data()` and `css_receive_data()` can partially send/recv due to kernel buffer limits. `css_readn()` and `css_writen()` in `connection_support.cpp` loop until the full count is satisfied or an error occurs. Callers must **not** assume a single `send()`/`recv()` completes the transfer.

## Packet Types (css_packet_type)

| Value | Name | Direction | Purpose |
|-------|------|-----------|---------|
| 1 | `COMMAND_TYPE` | Client→Server | RPC request with function_code + optional data |
| 2 | `DATA_TYPE` | Either | Data payload matching a prior COMMAND request_id |
| 3 | `ABORT_TYPE` | Client→Server | Cancel a pending request |
| 4 | `CLOSE_TYPE` | Either | Graceful connection close |
| 5 | `ERROR_TYPE` | Server→Client | Error response for a request |

## Request ID and Entry ID Encoding

Each `CSS_CONN_ENTRY` maintains a 16-bit `request_id` counter. The combined 32-bit EID (entry+request) is constructed as:

```c
#define CSS_RID_FROM_EID(eid)       ((unsigned short) LOW16BITS(eid))   // request_id
#define CSS_ENTRYID_FROM_EID(eid)   ((unsigned short) HIGH16BITS(eid))  // connection slot index
```

This allows the server to demultiplex concurrent in-flight requests on a single connection (upper bits = which `css_Conn_array` slot; lower bits = which pending request).

## Header Flags

| Flag Bit | Name | Meaning |
|----------|------|---------|
| `0x4000` | `NET_HEADER_FLAG_METHOD_MODE` | Connection is being used for a method/SP callback |
| `0x8000` | `NET_HEADER_FLAG_INVALIDATE_SNAPSHOT` | Server should invalidate the MVCC snapshot after this call |

## Server-Side Packet Queuing

When the server receives a packet from a client, it is classified and placed on one of several per-connection queues in `CSS_CONN_ENTRY`:

| Queue | Packet Type | Reader |
|-------|------------|--------|
| `request_queue` | `COMMAND_TYPE` | `css_queue_command_packet()` |
| `data_queue` | `DATA_TYPE` | `css_queue_data_packet()` — wakes waiting thread if one exists |
| `abort_queue` | `ABORT_TYPE` | Marks request as aborted |
| `error_queue` | `ERROR_TYPE` | Error from server to report upstream |
| `buffer_queue` | Pre-allocated | Server-allocated buffers waiting for data |
| `data_wait_queue` | — | Threads blocking in `css_return_queued_data_timeout()` |

## CSS Error Codes (css_error_code)

Internal error codes returned by CSS functions (distinct from `error_code.h`):

| Code | Name | Meaning |
|------|------|---------|
| 1 | `NO_ERRORS` | Success |
| 2 | `CONNECTION_CLOSED` | fd is closed |
| 4 | `ERROR_ON_READ` | `recv()` failed |
| 5 | `ERROR_ON_WRITE` | `send()` failed |
| 6 | `RECORD_TRUNCATED` | Received fewer bytes than header claimed |
| 8 | `READ_LENGTH_MISMATCH` | Header size != actual data size |
| 14 | `INTERRUPTED_READ` | `EINTR` during read |
| 17 | `TIMEDOUT_ON_QUEUE` | Wait on data queue expired |
| 18 | `INTERNAL_CSS_ERROR` | Internal inconsistency |

## Connection Status Codes (css_conn_status)

| Code | Name | Meaning |
|------|------|---------|
| 1 | `CONN_OPEN` | Active and usable |
| 2 | `CONN_CLOSED` | Closed, connection entry being freed |
| 3 | `CONN_CLOSING` | Draining in-flight requests before close |

## Function Code Dispatch

`NET_HEADER.function_code` determines which server-side handler is invoked. The mapping lives in `network.h` (the `NET_SERVER_*` enum). `css_internal_request_handler()` in `server_support.c` reads the function code from the queued command packet and dispatches to `css_Server_request_handler` (a registered function pointer set at server startup).

## Peer-Alive Check

`css_peer_alive(sd, timeout_ms)` in `tcp.c` is an application-level liveness probe. It tries to connect to port 7 (ECHO) of the peer host using a non-blocking `connect()` + `poll()`:
- If connect succeeds or is refused immediately — peer host is up.
- If `poll()` times out — peer is considered unreachable.

This is used by the `PRM_ID_CHECK_PEER_ALIVE` parameter (modes: `NONE`, `SERVER_ONLY`, `CLIENT_ONLY`, `BOTH`).

## Related

- [[components/connection|connection]] — full layer overview
- [[components/tcp-layer|tcp-layer]] — underlying socket operations
- [[components/cub-master|cub-master]] — master-level command codes
