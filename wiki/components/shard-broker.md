---
type: component
parent_module: "[[modules/broker|broker]]"
path: "src/broker/"
status: developing
purpose: "Shard broker variant — optional proxy layer that routes queries to multiple CUBRID shard databases based on a key-range or key-hash routing function"
key_files:
  - "shard_proxy.c — shard proxy process: lifecycle, shm init, cleanup"
  - "shard_proxy_handler.c — request routing logic, client context management"
  - "shard_metadata.c — routing metadata: shard_user/shard_range/shard_conn tables"
  - "shard_key_func.c — built-in MODULAR key function; dlopen() for custom fn"
  - "shard_shm.c/h — shard-specific shared memory helpers"
  - "shard_statement.c — prepared statement caching across shard CAS pool"
  - "shard_io.c / shard_io_posix.c — async I/O for shard proxy connections"
  - "broker_shm.h — T_SHM_PROXY, T_PROXY_INFO, T_SHARD_KEY, T_SHARD_CONN structs"
tags:
  - component
  - cubrid
  - shard
  - proxy
  - sharding
  - routing
related:
  - "[[components/broker-impl|broker-impl]]"
  - "[[components/broker-shm|broker-shm]]"
  - "[[components/cas|cas]]"
  - "[[modules/broker|broker]]"
  - "[[sources/cubrid-src-broker|cubrid-src-broker]]"
created: 2026-04-23
updated: 2026-04-23
---

# Shard Broker (`shard_proxy.c` and `shard_*`)

The **shard broker** is an optional additional process tier that sits between the broker listener and the CAS worker pool. When a broker is configured with `SHARD=ON`, the broker spawns one or more **shard proxy** processes. Clients still connect to the same broker port, but requests are transparently routed to a CUBRID shard database (one of N `cub_server` instances) based on a **shard key** extracted from the SQL query.

## Architecture

```
Client App
    │  TCP  (same BROKER_PORT as non-shard)
    ▼
Broker process (broker.c)
    │  accept() → shard_dispatch_thr_f
    │  Unix socket fd-passing
    ▼
Shard Proxy process (shard_proxy.c)   ← up to 8 proxy processes
    │  proxy_handler (shard_proxy_handler.c)
    │  shard key extraction + routing
    ├─ shard 0: CAS pool (cas.c, shard_cas_main)
    │             └─ cub_server_0
    ├─ shard 1: CAS pool
    │             └─ cub_server_1
    └─ shard N: CAS pool
                  └─ cub_server_N
```

The shard proxy processes communicate with the broker and with CAS pools through **shared memory** segments: `T_SHM_BROKER` (global), `T_SHM_APPL_SERVER` (per-shard CAS pool), and `T_SHM_PROXY` (proxy-level coordination). See [[components/broker-shm|broker-shm]].

## Routing Metadata

Shard routing tables are stored in three special CUBRID tables (queried at startup by `shard_metadata.c`):

| Table | Struct | Contents |
|-------|--------|---------|
| `shard_user` | `T_SHM_SHARD_USER` | db_name / db_user / db_password per shard user |
| `shard_range` | `T_SHM_SHARD_KEY` → `T_SHARD_KEY_RANGE` | key_column, [min, max) → shard_id ranges |
| `shard_conn` | `T_SHM_SHARD_CONN` | shard_id → (db_name, db_host) mapping |

All three are loaded into `T_SHM_PROXY` at startup and accessible by all proxy processes without further DB queries. Up to:
- 4 shard users (`MAX_SHARD_USER`)
- 2 key definitions (`MAX_SHARD_KEY`), each with up to 256 ranges (`SHARD_KEY_RANGE_MAX`)
- 256 shard connections (`MAX_SHARD_CONN`)

## Key Routing Function

The shard key function determines which shard a query targets. Two modes:

**Built-in MODULAR** (`shard_key_modular`): `shard_id = key_value % num_shard`. Simple hash sharding.

**Custom function** (dlopen): Loaded at startup from `shard_key_library_name` / `shard_key_function_name` in `T_SHM_PROXY`. The proxy calls `dlopen()` + `dlsym()` to load a user-supplied routing function. This allows arbitrary range mapping or consistent hashing.

Key extraction happens in `shard_proxy_handler.c`: the proxy parses the SQL text looking for `/*+ shard_key */` hints or the designated key column in WHERE clauses.

## `shard_proxy.c` — Proxy Process Lifecycle

```
shard_proxy.c: main()
  → proxy_shm_initialize()
       attach T_SHM_APPL_SERVER (per-shard CAS pools)
       attach T_SHM_PROXY (proxy coordination)
       attach T_SHM_SHARD_USER/KEY/CONN
  → proxy_handler_initialize()    # set up client context array
  → proxy_io_initialize()         # set up async I/O (epoll / kqueue)
  → shard_stmt_initialize()       # prepared statement cache
  → proxy_main_loop()

proxy_main_loop:
  ENDLESS loop {
    proxy_io_process()            # async I/O events: client + shard CAS fds
    proxy_handler_process()       # route pending requests
    proxy_set_hang_check_time()   # update claimed_alive_time in T_PROXY_INFO
  }

on shutdown:
  proxy_term()
    proxy_handler_destroy()
    proxy_io_destroy()
    shard_stmt_destroy()
    proxy_log_close() / proxy_access_log_close()
```

## Shard CAS (`shard_cas_main`)

The CAS processes attached to a shard proxy run `shard_cas_main()` instead of `cas_main()`. The differences:

- CAS connects to the broker via a **named Unix socket** (`port_name` in `T_SHM_APPL_SERVER`) rather than receiving a direct client fd
- `cas_register_to_proxy(proxy_sock_fd)` registers CAS with the proxy on startup
- `net_read_process()` reads through a `MSG_HEADER` wrapper added by the proxy layer
- Otherwise the same `server_fn_table[]` dispatch and `cas_execute.c` path apply

## `T_PROXY_INFO` — Per-Proxy Status

```c
struct t_proxy_info {
  int  proxy_id, pid;
  int  service_flag;       // SERVICE_ON / SERVICE_OFF
  int  max_shard, max_client, cur_client, max_context;
  int  wait_timeout;
  int  max_prepared_stmt_count;
  char ignore_shard_hint;  // route all queries to shard 0 (debug)
  char port_name[PATH_MAX];  // Unix socket name for CAS registration
  bool fixed_shard_user;

  // per-proxy statistics
  INT64 num_hint_key_queries_processed;
  INT64 num_hint_id_queries_processed;
  INT64 num_hint_none_queries_processed;  // queries with no shard hint
  INT64 num_hint_err_queries_processed;

  INT64 num_request_stmt;
  INT64 num_request_stmt_in_pool;        // statement cache hits
  INT64 num_connect_requests;
  INT64 num_restarts;

  time_t claimed_alive_time;             // broker hang-check reference

  T_SHARD_INFO    shard_info[SHARD_INFO_SIZE_LIMIT];  // per-shard CAS pool stats
  T_CLIENT_INFO   client_info[CLIENT_INFO_SIZE_LIMIT + 256]; // per-client state
};
```

## Limitations and Gotchas

- The shard feature is an **optional overlay** — most `shard_*` code paths are dead unless `SHARD=ON` in `cubrid_broker.conf`.
- `br_shard_flag` global in `broker.c` controls which dispatch path is taken (`shard_broker_process()` vs standard `broker_process()`).
- Prepared statement caching in `shard_statement.c` is across the entire shard CAS pool — the proxy must match driver-side statement handles to the correct shard's CAS.
- `ignore_shard_hint=ON` disables routing and sends all queries to shard 0 — useful for testing but not production.
- Up to 8 proxy processes (`MAX_PROXY_NUM`) can run per broker instance, for load distribution.
- `PROXY_RESERVED_FD` (256 on Linux, 128 elsewhere) is subtracted from max client count to ensure the proxy has headroom for internal file descriptors (log files, shard CAS connections, etc.).

## Integration

- [[components/broker-impl|broker-impl]] spawns proxy processes via `run_proxy_server()` and monitors them via `proxy_monitor_thr_f`
- [[components/broker-shm|broker-shm]] documents the `T_SHM_PROXY` segment and shard-specific structs
- [[components/cas|cas]] describes the shard CAS variant (`shard_cas_main`)

## Related

- Hub: [[components/broker-impl|broker-impl]]
- Source: [[sources/cubrid-src-broker|cubrid-src-broker]]
