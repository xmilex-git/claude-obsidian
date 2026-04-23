---
type: component
parent_module: "[[modules/src|src]]"
path: "src/connection/"
status: developing
purpose: "HA heartbeat protocol: node state machine (master/slave/replica), process registration with cub_master, failover detection timing, HBP packet format"
key_files:
  - "heartbeat.c / heartbeat.h (CSS-level send/recv, register/deregister, CS_MODE reader thread)"
  - "connection_defs.h (ha_mode, ha_server_mode, ha_log_applier_state enums)"
  - "server_support.c (HA server state machine: ha_Server_state, css_transit_ha_server_state)"
tags:
  - component
  - cubrid
  - ha
  - heartbeat
  - failover
related:
  - "[[components/connection|connection]]"
  - "[[components/cub-master|cub-master]]"
  - "[[components/network-protocol|network-protocol]]"
  - "[[components/thread|thread]]"
  - "[[Architecture Overview]]"
  - "[[sources/cubrid-src-connection|cubrid-src-connection]]"
created: 2026-04-23
updated: 2026-04-23
---

# HA Heartbeat

CUBRID's High Availability (HA) heartbeat subsystem tracks the liveness and role of every node in a replicated cluster. It is active only when `HA_MODE` is not `off`. The code lives in `heartbeat.c / heartbeat.h` and is driven by `cub_master` (the per-host coordinator) with help from the DB server's `server_support.c`.

## HA Modes (ha_mode)

| Mode | Value | Description |
|------|-------|-------------|
| `HA_MODE_OFF` | 0 | Standalone, no HA |
| `HA_MODE_FAIL_OVER` | 1 | Unused |
| `HA_MODE_FAIL_BACK` | 2 | Active fail-back |
| `HA_MODE_LAZY_BACK` | 3 | Not yet implemented |
| `HA_MODE_ROLE_CHANGE` | 4 | Role-change mode |
| `HA_MODE_REPLICA` | 5 | Read-only replica |

## Node States (HB_NODE_STATE)

The heartbeat daemon classifies each HA node into one of six states:

| State | Meaning |
|-------|---------|
| `HB_NSTATE_UNKNOWN` | Not yet determined |
| `HB_NSTATE_SLAVE` | Standby — replicating from master |
| `HB_NSTATE_TO_BE_MASTER` | Elected, transitioning to active |
| `HB_NSTATE_TO_BE_SLAVE` | Transitioning back to standby |
| `HB_NSTATE_MASTER` | Active primary node |
| `HB_NSTATE_REPLICA` | Read-only replica (never becomes master) |

## HA Process Types (HB_PROC_TYPE)

Three process types register with cub_master and participate in heartbeat:

| Type | String | Role |
|------|--------|------|
| `HB_PTYPE_SERVER` | `"HA-server"` | The cub_server database process |
| `HB_PTYPE_COPYLOGDB` | `"HA-copylogdb"` | Copies WAL log from primary |
| `HB_PTYPE_APPLYLOGDB` | `"HA-applylogdb"` | Applies copied logs on standby |

## Heartbeat Timing Constants (from heartbeat.h)

| Constant | Default | Purpose |
|----------|---------|---------|
| `HB_DEFAULT_HEARTBEAT_INTERVAL_IN_MSECS` | 500 ms | Inter-beat period |
| `HB_DEFAULT_CALC_SCORE_INTERVAL_IN_MSECS` | 3 s | How often node scores are recomputed |
| `HB_DEFAULT_FAILOVER_WAIT_TIME_IN_MSECS` | 3 s | Grace period before triggering failover |
| `HB_DEFAULT_MAX_HEARTBEAT_GAP` | 5 | Missed beats before a node is considered dead |
| `HB_DEFAULT_CHANGEMODE_INTERVAL_IN_MSECS` | 5 s | HA mode-change cooldown |
| `HB_DEFAULT_HA_PORT_ID` | 59901 | UDP port for inter-node heartbeat messages |
| `HB_DEFAULT_UNACCEPTABLE_PROC_RESTART_TIMEDIFF_IN_MSECS` | 2 min | Max restart frequency before giving up |

> [!key-insight] Failover decision path
> A node declares a master dead only after `HB_DEFAULT_MAX_HEARTBEAT_GAP` (5) consecutive missed 500 ms heartbeats — roughly 2.5 seconds of silence — and then waits an additional `HB_DEFAULT_FAILOVER_WAIT_TIME_IN_MSECS` (3 s) before initiating role transition. Total worst-case detection: ~5.5 s from last heard beat to new master elected.

## HBP Packet Format

Inter-node heartbeat UDP messages use `HBP_HEADER`:

```c
struct hbp_header
{
  unsigned char type;     // HBP_CLUSTER_MESSAGE (0 = HBP_CLUSTER_HEARTBEAT)
  char reserved:7;
  char r:1;               // is_request flag
  unsigned short len;     // total message length
  unsigned int seq;       // sequence number
  char group_id[64];      // HA group identifier
  char orig_host_name[MAXHOSTNAMELEN];
  char dest_host_name[MAXHOSTNAMELEN];
};
```

Process registration messages use `HBP_PROC_REGISTER`:

```c
struct hbp_proc_register
{
  int pid;                              // network byte order
  int type;                             // HB_PROC_TYPE (network byte order)
  char exec_path[128];
  char args[16 * 64];                   // argv joined by spaces
};
```

## Registration Protocol (CSS Channel)

HA processes communicate with cub_master over their existing **CSS connection** (not the UDP heartbeat channel) using `css_send_heartbeat_request()` / `css_send_heartbeat_data()` — simple `send()` calls without `NET_HEADER` framing:

1. Process calls `hb_register_to_master(conn, type)`:
   - Sends `SERVER_REGISTER_HA_PROCESS` as a 4-byte network-order integer.
   - Sends `HBP_PROC_REGISTER` struct as data.
2. On shutdown: `hb_deregister_from_master()` sends `SERVER_DEREGISTER_HA_PROCESS` + PID.

> [!note] No NET_HEADER on heartbeat channel
> Heartbeat request/data functions bypass `NET_HEADER` framing entirely. The receiver uses `css_readn()` directly. This is intentional — heartbeat is a side channel, not a regular CSS RPC.

## CS_MODE Reader Thread

When a client-mode HA process (`CS_MODE`) connects to cub_master, it spawns `hb_thread_master_reader` — a dedicated thread that blocks in `hb_process_master_request()`, reading commands from the master. If the master link breaks, the thread calls `hb_process_term()` and sends `SIGTERM` to itself, triggering a clean shutdown.

## Server-Side HA State Machine (server_support.c)

The server maintains `ha_Server_state` (`HA_SERVER_STATE`):

| State | Meaning |
|-------|---------|
| `HA_SERVER_STATE_IDLE` | Starting, not yet active |
| `HA_SERVER_STATE_ACTIVE` | Primary: accepts read+write |
| `HA_SERVER_STATE_STANDBY` | Standby: read-only |
| `HA_SERVER_STATE_BACKUP` | Backup replica |

`css_transit_ha_server_state()` validates legal transitions. The server also tracks up to 5 `applylogdb` clients in `ha_Log_applier_state[]` — all must reach `HA_LOG_APPLIER_STATE_WORKING` before the server can report itself as fully operational.

## Log Applier States (ha_log_applier_state)

| State | Meaning |
|-------|---------|
| `HA_LOG_APPLIER_STATE_NA` | Not available (slot unused) |
| `HA_LOG_APPLIER_STATE_UNREGISTERED` | Registered, not yet active |
| `HA_LOG_APPLIER_STATE_RECOVERING` | Catching up with log backlog |
| `HA_LOG_APPLIER_STATE_WORKING` | Fully caught up, applying live logs |
| `HA_LOG_APPLIER_STATE_DONE` | Finished (shutdown) |
| `HA_LOG_APPLIER_STATE_ERROR` | Applier hit a fatal error |

## Related

- [[components/connection|connection]] — CSS layer that carries heartbeat registration messages
- [[components/cub-master|cub-master]] — receives registrations, drives HA job scheduler
- [[components/thread|thread]] — `hb_thread_master_reader` runs as a cubthread daemon
