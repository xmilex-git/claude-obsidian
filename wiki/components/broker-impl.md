---
type: component
parent_module: "[[modules/broker|broker]]"
path: "src/broker/"
status: developing
purpose: "Connection broker (multi-process router) and CAS (CUBRID Application Server) worker processes — the middle tier between client drivers and cub_server"
key_files:
  - "broker.c — main broker process: port listener, CAS lifecycle, dispatch threads"
  - "cas.c — CAS worker process entry point, protocol function table, main loop"
  - "cas_execute.c — SQL execution within CAS: prepare/execute/fetch"
  - "cas_function.c — CAS protocol function dispatch table"
  - "cas_network.c / cas_net_buf.c — CAS network I/O and output buffer"
  - "cas_handle.c — connection and statement handle management"
  - "broker_shm.c/h — shared memory IPC: T_SHM_BROKER, T_SHM_APPL_SERVER"
  - "broker_config.c — cubrid_broker.conf parsing"
  - "broker_monitor.c — broker_monitor utility"
  - "broker_admin.c — cubrid broker admin commands"
  - "broker_acl.c — IP/user access control list"
  - "broker_log_util.c / broker_log_top.c — slow/SQL log analysis"
  - "shard_proxy.c — shard proxy process (optional sharding layer)"
  - "shard_metadata.c — shard routing metadata (shard_user/range/conn tables)"
public_api:
  - "broker_shm: uw_shm_open / uw_shm_create / uw_shm_destroy / uw_shm_detach"
  - "uw_sem_init / uw_sem_wait / uw_sem_post — POSIX semaphore wrappers"
  - "CON_STATUS_LOCK / CON_STATUS_UNLOCK — spinlock-or-semaphore guarding uts_status"
  - "cas_main_loop(ops) — common CAS main loop in cas_common_main.c"
tags:
  - component
  - cubrid
  - broker
  - cas
  - connection-pooling
  - shared-memory
  - multi-process
related:
  - "[[modules/broker|broker]]"
  - "[[components/cas|cas]]"
  - "[[components/broker-shm|broker-shm]]"
  - "[[components/shard-broker|shard-broker]]"
  - "[[components/connection|connection]]"
  - "[[components/cub-master|cub-master]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[components/tcp-layer|tcp-layer]]"
  - "[[Architecture Overview]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[sources/cubrid-src-broker|cubrid-src-broker]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/broker/` — Connection Broker & CAS Worker Processes

> [!note] Scope disambiguation
> This page covers the **implementation** in `src/broker/`. The top-level [[modules/broker|broker]] page describes only the CMake target that wraps this code. See [[modules/broker|broker]] for build-level context.

`src/broker/` implements the **middle tier** of CUBRID's three-tier architecture: the broker process routes client connections to a pool of CAS (CUBRID Application Server) worker processes, which in turn open sessions with `cub_server`. All inter-process coordination happens through **shared memory** — the broker and its CAS children never communicate via sockets.

## Three-Tier Topology

```
Client App (JDBC/ODBC/CCI driver)
        │  TCP port (cubrid_broker.conf: BROKER_PORT)
        ▼
┌──────────────────────────────────┐
│  broker process (broker.c)       │  one per broker definition
│  receiver_thr_f   → accept()     │
│  dispatch_thr_f   → find_idle_cas│
│  cas_monitor_thr_f               │
│  hang_check_thr_f                │
│  server_monitor_thr_f            │
└─────────────┬────────────────────┘
              │  shared memory (T_SHM_APPL_SERVER)
              │  socket-fd passing (broker_send_fd.c)
              ▼
┌──────────────────────────────────┐
│  CAS processes (cas.c)           │  one per concurrent client session
│  cas_main_loop()                 │
│  server_fn_table[func_code]()    │
│  → cas_execute.c (prepare/exec)  │
└─────────────┬────────────────────┘
              │  CSS protocol (src/connection/)
              ▼
┌──────────────────────────────────┐
│  cub_server process              │
│  (via css_connect_to_cubrid_server)│
└──────────────────────────────────┘
```

Optional **shard proxy** layer sits between broker and CAS when `SHARD` mode is enabled — see [[components/shard-broker|shard-broker]].

## Broker Process (`broker.c`)

The broker is a **single multi-threaded process** that owns the TCP listener socket. Its main threads:

| Thread function | Role |
|----------------|------|
| `receiver_thr_f` | `accept()` incoming connections; push to job queue |
| `dispatch_thr_f` | Pop job queue; call `find_idle_cas()`; pass fd via `broker_send_fd` |
| `cas_monitor_thr_f` | Detect dead/hung CAS; restart via `run_appl_server()` |
| `psize_check_thr_f` | Monitor CAS process memory (PSIZE) against `appl_server_max_size` |
| `hang_check_thr_f` | Compare `claimed_alive_time` in shared memory; kill frozen CAS |
| `server_monitor_thr_f` | Poll `cub_server` state via `css_connect_to_master_server()` |
| `proxy_monitor_thr_f` | Monitor shard proxy processes (shard mode only) |

### CAS Lifecycle (from broker perspective)

```
broker startup
  → broker_init_shm()         # create T_SHM_BROKER + T_SHM_APPL_SERVER
  → run_appl_server()         # fork/exec cub_cas for each min_appl_server slot
  → shm_appl->as_info[i].uts_status = UTS_STATUS_IDLE

client connects
  → receiver_thr_f: accept() → push (clt_sock_fd, ip) to job queue
  → dispatch_thr_f: find_idle_cas() → returns as_index of IDLE CAS
  → CON_STATUS_LOCK; set uts_status = UTS_STATUS_BUSY
  → broker_send_fd: pass clt_sock_fd to CAS over Unix socket
  → CON_STATUS_UNLOCK

dynamic scaling
  → broker_add_new_cas()      # if all CAS busy and < appl_server_max_num
  → broker_drop_one_cas_by_time_to_kill()  # idle > time_to_kill → SERVICE_OFF_ACK
```

### Key Global State

```c
static T_SHM_BROKER      *shm_br   = NULL;  // broker-level shared memory
static T_SHM_APPL_SERVER *shm_appl = NULL;  // CAS slot array + job queue
static T_BROKER_INFO      *br_info_p = NULL;
static T_SHM_PROXY        *shm_proxy_p = NULL; // shard proxy shm (if shard mode)
```

## CAS Worker Process (`cas.c`)

Each CAS process is a **separate OS process** (fork from broker on Unix; separate exec on Windows). CAS lifecycle:

```
cas.c: main()
  → cas_init()                # parse args, open shared memory segment
  → cas_init_shm()            # attach T_SHM_APPL_SERVER, find own as_info slot
  → cas_main() or shard_cas_main()
       └─ cas_main_loop(&ops) # common loop in cas_common_main.c

cas_main_loop:
  1. Wait for broker to send client fd (recv via broker_send_fd)
  2. cas_db_connect() → css_connect_to_cubrid_server()
  3. cas_post_db_connect() → update shm_appl->as_info[idx] stats
  4. Request loop:
       read MSG_HEADER from client socket
       dispatch via server_fn_table[func_code]()
       write response via cas_net_buf → cas_network
  5. Transaction ends → CON_STATUS_OUT_TRAN
  6. Keep-alive: if keep_connection=ON, wait for next request
     else: close → shm uts_status = UTS_STATUS_IDLE
  7. On session end: cas_cleanup_session()
```

### CAS Function Dispatch Table

`server_fn_table[]` in `cas.c` maps `CAS_FC_*` codes to handler functions (all in `cas_function.c` / `cas_execute.c`):

| Code | Handler | Purpose |
|------|---------|---------|
| `CAS_FC_PREPARE` | `fn_prepare` | Parse + prepare SQL |
| `CAS_FC_EXECUTE` | `fn_execute` | Execute prepared statement |
| `CAS_FC_FETCH` | `fn_fetch` | Fetch result rows |
| `CAS_FC_END_TRAN` | `fn_end_tran` | Commit or rollback |
| `CAS_FC_SCHEMA_INFO` | `fn_schema_info` | Catalog / schema queries |
| `CAS_FC_LOB_NEW/WRITE/READ` | `fn_lob_*` | LOB operations |
| `CAS_FC_XA_PREPARE/RECOVER` | `fn_xa_*` | XA distributed transactions |
| `CAS_FC_CURSOR_*` | `fn_cursor*` | Positioned updates, cursor ops |
| `CAS_FC_SET_CAS_CHANGE_MODE` | `fn_set_cas_change_mode` | CAS reuse mode |

SQL execution (`fn_execute`) calls into `cas_execute.c`, which calls the client API (`db_*` functions in `src/compat/`), which routes to `cub_server` over the CSS connection established during `cas_db_connect`.

## Shared Memory Architecture

See [[components/broker-shm|broker-shm]] for full structure layout. Key relationship:

```
T_SHM_BROKER (shm key: DEFAULT_SHM_KEY = 0x3f5d1c0a)
  └─ T_BROKER_INFO br_info[1..N]   # one per broker name

T_SHM_APPL_SERVER (separate shm segment per broker)
  ├─ T_MAX_HEAP_NODE job_queue[4096+1]   # priority queue of waiting connections
  ├─ T_APPL_SERVER_INFO as_info[4096]    # one slot per possible CAS
  │    ├─ uts_status (IDLE/BUSY/RESTART/CON_WAIT/STOP)
  │    ├─ con_status (OUT_TRAN/IN_TRAN/CLOSE/CLOSE_AND_CONNECT)
  │    ├─ pid, psize, last_access_time, claimed_alive_time
  │    ├─ con_status_sem (POSIX semaphore — locks broker↔CAS status update)
  │    └─ counters: num_requests_received, num_queries_processed, ...
  └─ ACCESS_INFO access_info[50]          # ACL table
```

## Connection Pooling

CUBRID's "connection pooling" is **process-based**: each CAS process can serve multiple sequential client sessions without exiting (controlled by `keep_connection` config). The key states:

- `UTS_STATUS_IDLE` — CAS waiting, available for dispatch
- `UTS_STATUS_BUSY` — CAS serving a client
- `UTS_STATUS_CON_WAIT` — CAS has DB connection but waiting between requests (out-of-transaction)
- `UTS_STATUS_RESTART` — broker has signalled CAS to restart

The `CON_STATUS_LOCK` / `CON_STATUS_UNLOCK` macros guard the `con_status` / `uts_status` transition to prevent the broker and CAS from reading stale state simultaneously (POSIX semaphore on Linux; Peterson's algorithm on Windows).

## Configuration

Parsed from `cubrid_broker.conf` by `broker_config.c`. Key parameters propagated into shared memory:

| Parameter | Shm field | Effect |
|-----------|-----------|--------|
| `MIN_NUM_APPL_SERVER` | `appl_server_min_num` | CAS processes always running |
| `MAX_NUM_APPL_SERVER` | `appl_server_max_num` | Hard cap (max 4096) |
| `APPL_SERVER_MAX_SIZE` | `appl_server_max_size` | Restart CAS if RSS exceeds (MB) |
| `TIME_TO_KILL` | `time_to_kill` | Idle CAS eviction timeout |
| `SESSION_TIMEOUT` | `session_timeout` | Client idle timeout |
| `KEEP_CONNECTION` | `keep_connection` | ON/OFF/AUTO connection reuse |
| `SQL_LOG` | `sql_log_mode` | Log all/error/short/off |
| `SLOW_LOG` | `slow_log_mode` | Log queries over long_query_time |

## Error Handling

CAS processes use a **separate error logging path** from the main server:
- `cas_error_log_write()` — writes to `*.err` file in `log_dir`
- **Not** `er_set()` — the CUBRID engine error convention does not apply in the broker tier
- Error codes are translated to driver protocol codes before sending to client

## Sub-Components

- [[components/cas|cas]] — CAS worker process: lifecycle, request loop, db connection
- [[components/broker-shm|broker-shm]] — shared memory layout: `T_SHM_BROKER`, `T_SHM_APPL_SERVER`, semaphore protocol
- [[components/shard-broker|shard-broker]] — optional shard proxy layer for database sharding

## Integration

- [[modules/broker|broker]] — top-level CMake target that builds this code
- [[components/connection|connection]] — CAS uses `css_connect_to_cubrid_server()` to reach `cub_server`
- [[components/cub-master|cub-master]] — broker queries master to check server liveness (`server_monitor_thr_f`)
- [[components/network-protocol|network-protocol]] — CSS packet framing used by CAS↔server connection
- [[components/tcp-layer|tcp-layer]] — raw socket primitives used by broker listener
- [[Architecture Overview]] — overall client → broker → CAS → DB server topology
- [[Build Modes (SERVER SA CS)]] — broker/CAS always compile as standalone processes, not server/SA/CS modes

## Related

- Parent target: [[modules/broker|broker]]
- Source: [[sources/cubrid-src-broker|cubrid-src-broker]]
