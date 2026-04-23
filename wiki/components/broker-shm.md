---
type: component
parent_module: "[[modules/broker|broker]]"
path: "src/broker/"
status: developing
purpose: "Shared memory IPC layer between broker process and CAS worker processes — T_SHM_BROKER, T_SHM_APPL_SERVER, job queue, per-CAS status, semaphore locking"
key_files:
  - "broker_shm.h — all shared memory struct definitions and CON_STATUS_LOCK macros"
  - "broker_shm.c — uw_shm_open / uw_shm_create / uw_shm_destroy; sem wrappers"
tags:
  - component
  - cubrid
  - shared-memory
  - ipc
  - broker
related:
  - "[[components/broker-impl|broker-impl]]"
  - "[[components/cas|cas]]"
  - "[[components/shard-broker|shard-broker]]"
  - "[[modules/broker|broker]]"
  - "[[sources/cubrid-src-broker|cubrid-src-broker]]"
created: 2026-04-23
updated: 2026-04-23
---

# Broker Shared Memory (`broker_shm.c/h`)

The broker and all its CAS child processes coordinate exclusively through **POSIX shared memory** segments. No socket or pipe is used for status and configuration exchange — only the initial client socket fd is passed over a Unix socket. This design keeps the broker's hot path out of the kernel syscall overhead of pipes/signals.

## Segment Hierarchy

Three distinct shared memory segments exist per broker instance:

| Segment type | Struct | Who creates | Key source |
|-------------|--------|-------------|-----------|
| Broker-global | `T_SHM_BROKER` | `cub_broker` admin | `DEFAULT_SHM_KEY` (0x3f5d1c0a) |
| Per-broker CAS pool | `T_SHM_APPL_SERVER` | broker process | derived from broker name |
| Shard proxy | `T_SHM_PROXY` | broker (shard mode) | derived from proxy name |

`uw_shm_open(shm_key, which_shm, shm_mode)` attaches an existing segment; `uw_shm_create(shm_key, size, which_shm)` creates a new one. `which_shm` is one of `SHM_BROKER`, `SHM_APPL_SERVER`, `SHM_PROXY`.

> [!warning] No pointers in shared memory
> `broker_shm.h` explicitly warns: "Be sure not to include any pointer type in shared memory segment since the processes will not care where the shared memory segment is attached." All inter-process references are by integer index, not pointer.

## `T_SHM_BROKER` — Global Broker Registry

```c
struct t_shm_broker {
  int           magic;               // version check
  unsigned char my_ip_addr[4];
  uid_t         owner_uid;           // Unix only
  int           num_broker;
  char          admin_log_file[PATH_MAX];
  char          access_control_file[PATH_MAX];
  bool          access_control;
  bool          acl_default_policy;  // allow or deny unlisted brokers
  T_BROKER_INFO br_info[1];          // flexible array: one per broker name
};
```

`T_BROKER_INFO` holds per-broker config (port, min/max CAS count, timeouts, SSL mode, etc.) and runtime counters. The magic number encodes the CUBRID version: `MAJOR * 1000000 + MINOR * 10000 + SEQ`.

## `T_SHM_APPL_SERVER` — CAS Pool Segment

The largest and most-accessed segment. Layout:

```
T_SHM_APPL_SERVER
  ├─ config scalars: broker_port, session_timeout, query_timeout,
  │    sql_log_mode, keep_connection, access_mode, ...
  ├─ broker_name[BROKER_NAME_LEN], appl_server_name[32]
  ├─ ACCESS_INFO access_info[50]        # ACL: dbname + dbuser + IP list
  ├─ T_MAX_HEAP_NODE job_queue[4097]    # max-heap of pending connections
  │    (job_queue[0].id = queue size counter)
  ├─ T_SHARD_CONN_INFO shard_conn_info[SHARD_INFO_SIZE_LIMIT]  # shard only
  ├─ T_APPL_SERVER_INFO as_info[4096]   # one slot per possible CAS process
  └─ T_DB_SERVER unusable_databases[2][200]  # HA unusable DB tracking
```

### `T_APPL_SERVER_INFO` — Per-CAS Status Record

This is the critical inter-process communication struct — both broker and CAS read/write it:

| Field | Type | Owner | Purpose |
|-------|------|-------|---------|
| `uts_status` | `char` | both | IDLE / BUSY / RESTART / START / CON_WAIT / STOP |
| `con_status` | `char` | CAS | OUT_TRAN / IN_TRAN / CLOSE / CLOSE_AND_CONNECT |
| `pid` | `int` | broker | CAS OS process ID |
| `psize` | `int` | broker | CAS resident set size (pages) |
| `service_flag` | `char` | broker | SERVICE_ON / SERVICE_OFF / SERVICE_OFF_ACK |
| `reset_flag` | `char` | broker | Signal CAS to reset state |
| `claimed_alive_time` | `time_t` | CAS | Updated in hot loop; broker hang-detector reads |
| `last_access_time` | `time_t` | CAS | Last client request; broker uses for time_to_kill |
| `transaction_start_time` | `time_t` | CAS | When current transaction began |
| `fn_status` | `int` | CAS | Current `CAS_FC_*` code |
| `con_status_sem` | `sem_t` | init | POSIX semaphore protecting uts/con_status transition |
| `num_requests_received` | `INT64` | CAS | Cumulative request counter |
| `num_queries_processed` | `INT64` | CAS | Cumulative query counter |
| `num_long_queries` | `INT64` | CAS | Queries exceeding `long_query_time` |
| `num_error_queries` | `INT64` | CAS | Failed queries |
| `cas_clt_ip[4]` | `unsigned char` | CAS | Connected client IPv4 |
| `driver_version` | `char[]` | CAS | Client driver version string |
| `log_msg` | `char[256]` | CAS | Last log message (broker_monitor display) |

### `UTS_STATUS_*` State Machine

```
UTS_STATUS_START
  → (CAS attaches shm, sets IDLE)
UTS_STATUS_IDLE
  ←→ (broker dispatches client fd) →
UTS_STATUS_BUSY
  → (client disconnects, keep_connection=OFF) → process exits → broker restarts
  → (client disconnects, keep_connection=ON) → UTS_STATUS_IDLE
  → (broker sends SERVICE_OFF_ACK) → UTS_STATUS_STOP → process exits
UTS_STATUS_CON_WAIT
  → CAS has DB connection open, waiting between sessions (out-of-transaction)
UTS_STATUS_RESTART
  → broker has decided to restart this CAS slot
```

## `CON_STATUS_LOCK` — Concurrent State Update Protocol

The broker (dispatch thread) and CAS (main loop) both need to atomically check + update `uts_status` and `con_status`. The lock mechanism differs by platform:

**Linux/macOS** — POSIX semaphore (`sem_t con_status_sem` in shared memory):
```c
#define CON_STATUS_LOCK(AS_INFO, LOCK_OWNER)   uw_sem_wait(&(AS_INFO)->con_status_sem)
#define CON_STATUS_UNLOCK(AS_INFO, LOCK_OWNER) uw_sem_post(&(AS_INFO)->con_status_sem)
```

**Windows** — Peterson's mutual exclusion algorithm:
```c
// con_status_lock[0] = broker interest flag
// con_status_lock[1] = CAS interest flag
// con_status_lock_turn = who yields if both want in
#define CON_STATUS_LOCK(AS_INFO, LOCK_OWNER) \
  (AS_INFO)->con_status_lock[LOCK_OWNER] = TRUE; \
  (AS_INFO)->con_status_lock_turn = LOCK_WAITER; \
  while (both interested && turn == LOCK_WAITER) SLEEP_MILISEC(0,10);
```

`LOCK_OWNER` is either `CON_STATUS_LOCK_BROKER` (0) or `CON_STATUS_LOCK_CAS` (1).

## `T_SHM_PROXY` — Shard Proxy Segment

When shard mode is active, a third segment stores the shard proxy configuration. See [[components/shard-broker|shard-broker]] for details. Key fields:

```c
struct t_shm_proxy {
  int           num_proxy;
  int           max_client, max_context;
  int           shard_key_modular;
  char          shard_key_library_name[PATH_MAX]; // dlopen for custom key fn
  char          shard_key_function_name[PATH_MAX];
  T_SHM_SHARD_USER  shm_shard_user;   // user→shard_db mapping
  T_SHM_SHARD_CONN  shm_shard_conn;   // shard_id → (db_name, host)
  T_SHM_SHARD_KEY   shm_shard_key;    // key_column + range[] → shard_id
  T_PROXY_INFO  proxy_info[8];        // up to 8 proxy processes
};
```

## Shared Memory API

| Function | Purpose |
|---------|---------|
| `uw_shm_open(key, which, mode)` | Attach existing segment (admin or monitor mode) |
| `uw_shm_create(key, size, which)` | Create new segment |
| `uw_shm_destroy(key)` | `shmctl(IPC_RMID)` |
| `uw_shm_detach(ptr)` | `shmdt()` |
| `uw_sem_init / wait / post / destroy` | POSIX semaphore wrappers (or Win32 named semaphore) |

## Integration

- [[components/broker-impl|broker-impl]] creates segments at startup and uses them for dispatch
- [[components/cas|cas]] attaches segments on startup; reads/writes `as_info[own_index]`
- [[components/shard-broker|shard-broker]] adds the `T_SHM_PROXY` segment when shard mode is on

## Related

- Hub: [[components/broker-impl|broker-impl]]
- Source: [[sources/cubrid-src-broker|cubrid-src-broker]]
