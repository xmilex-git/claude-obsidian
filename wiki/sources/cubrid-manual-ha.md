---
created: 2026-04-27
type: source
title: "CUBRID Manual — High Availability (ha.rst)"
source_path: "/home/cubrid/cubrid-manual/en/ha.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - ha
  - replication
  - failover
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-config-params]]"
  - "[[sources/cubrid-manual-error-codes]]"
  - "[[components/heartbeat]]"
  - "[[components/log-manager]]"
  - "[[components/recovery]]"
  - "[[components/cub-master]]"
---

# CUBRID Manual — High Availability (ha.rst)

**Ingested:** 2026-04-27
**Source file:** `/home/cubrid/cubrid-manual/en/ha.rst` (4194 lines — the second-largest manual file)
**Linux only.** No HA support on Windows.

## Section Map (top-level)

| Section | Content |
|---|---|
| **CUBRID HA Concept** | Nodes & groups (master / slave / replica), processes (cub_master, cub_server, copylogdb, applylogdb), server vs node status, heartbeat protocol, failover/failback, broker mode (RW/RO/SO) |
| **CUBRID HA Features** | Server duplexing, broker duplexing, log multiplexing (SYNC vs ASYNC), failover details |
| **Quick Start** | 1:1 master ↔ slave example: createdb, configure cubrid.conf + cubrid_ha.conf, configure databases.txt, start heartbeat, verify, configure broker `ACCESS_MODE`, JDBC `altHosts` URL |
| **Environment Configuration** | `cubrid.conf` (`ha_mode`, `ha_node_list`, `ha_replica_list`, log/checkpoint params), `cubrid_ha.conf` (`ha_port_id`, `ha_db_list`, `ha_copy_log_max_archives`, etc.), `cubrid_broker.conf` (`PREFERRED_HOSTS`, `RECONNECT_TIME`, `CONNECT_ORDER`, `MAX_NUM_DELAYED_HOSTS_LOOKUP`), `databases.txt` (db-host order), JDBC URL `altHosts`, `loadBalance`, `rcTime`, `connectTimeout` |
| **Building Replication** | 1:1, 1:N (replica fan-out), N:N, mixed master/replica scenarios |
| **Operations** | `cubrid heartbeat start/stop/status/reload`, `cubrid changemode`, `cubrid applyinfo`, `cubrid copylogdb`/`applylogdb` standalone, `cubrid restoreslave` |
| **HA Logging & Diagnostics** | `cub_master.err` events, `applylogdb.log` / `copylogdb.log`, `db_ha_apply_info` catalog table |
| **Failure scenarios** | Master crash → failover, slave crash, network partition (split-brain detection via ping_hosts), applier delay, OOM |
| **Performance Tuning** | `ha_log_applier_max_mem_size`, `ha_apply_max_mem_size`, copylogdb / applylogdb thread counts, network bandwidth |

## Key Concepts

### Node roles
- **Master**: read-write, source of truth.
- **Slave**: standby, read-only via Read Only / Standby Only brokers, **eligible for failover** to become master.
- **Replica**: read-only, **NOT eligible** for failover. Use case: read scaling, geo-distributed replicas.
- Configured via `ha_mode = on | off | replica` (cubrid.conf) and `ha_node_list` / `ha_replica_list` (cubrid_ha.conf).

### Per-node processes (in HA mode)
1. `cub_master` — heartbeat, failover orchestration
2. `cub_server` — DB server (active or standby)
3. `copylogdb` — fetches replication log from peer
4. `applylogdb` — applies replication log into local DB
5. `cub_broker` + CAS — front-end (typically separate machines)

### Heartbeat
- UDP on `ha_port_id` between every pair of `cub_master` processes in the group.
- Internal interval (not user-configurable).
- `ha_ping_hosts` (cubrid_ha.conf) — additional reachability targets to disambiguate "peer down" from "network down" (split-brain prevention).

### Failover
- Highest-priority slave becomes master automatically when current master fails.
- **No automatic failback** — operator must manually restore old master as slave.
- Score-based: master computes scores for all nodes, promotes highest.

### Broker modes
- **RW (Read Write)**: connects to active server. Falls through to standby temporarily; disconnects after each transaction to retry RW.
- **RO (Read Only)**: prefers standby. Can connect to active temporarily; rebinds after `RECONNECT_TIME` (default 600 s) or `cubrid broker reset`.
- **SO (Standby Only)**: standby only; no service if no standby.
- Connect order from `db-host` field in `databases.txt`.

### Log multiplexing
- **SYNC mode**: commit waits for slave to confirm log copy. Higher latency, no data loss.
- **ASYNC mode**: commit completes immediately; replication lag possible. Data inconsistency on master crash.
- Configured per-DB in `cubrid_ha.conf`.

## Notable cubrid.conf HA params (`config.rst:2249-2257`)
- `ha_mode` — off / on / replica. Cannot be changed dynamically.
- `force_remove_log_archives` — **MUST be `no`** under HA, else archive logs needed by replication may be deleted, causing inconsistency. Manual is explicit about this.
- `log_max_archives` — max archive logs to retain.

## Notable cubrid_ha.conf params
- `ha_port_id` — UDP port for heartbeat (between master processes).
- `ha_node_list` — comma-separated list of HA-eligible hostnames.
- `ha_replica_list` — comma-separated list of replica-only hostnames.
- `ha_db_list` — list of DBs that participate in HA (must match across nodes).
- `ha_copy_log_max_archives` — separate retention for replication archive logs.
- `ha_apply_max_mem_size` — old name; capped at 500 MB pre-9.x; renamed `ha_log_applier_max_mem_size` thereafter.
- `ha_log_applier_max_mem_size` — protects against unbounded growth; triggers error `-1035` and shutdown when exceeded or grown >2× quickly.
- `ha_delay_limit` / `ha_delay_limit_delta` — replication-delay alarm thresholds. Raises `-1025` (delay) / `-1026` (cleared).

## JDBC / CCI URL HA properties
- `altHosts` — comma-separated list of fallback brokers.
- `loadBalance=true` (or `rr`) for round-robin; `sh` for random shuffle.
- `rcTime` — reconnect-to-primary interval after failover.
- `connectTimeout` — initial connect timeout per host.
- ACCESS_MODE on alternate brokers is **ignored** by client-side selection — driver picks regardless.

## Operations
- **Startup order**: `cubrid heartbeat start` on the node intended to become master FIRST. The first node to start becomes master.
- **`cubrid heartbeat status`** — node state (master/slave/replica), peer reachability.
- **`cubrid changemode <db>`** — view/change server status (active/standby/maintenance). Maintenance mode = read-only and won't be elevated to active.
- **`cubrid applyinfo`** — replication progress and log info.
- **`cubrid restoreslave`** — re-create slave from master backup after divergence.

## Errors of note (`error_log_ha.rst`)
- **-898**: generic replication error
- **-970**: HA-mode change
- **-986/987**: heartbeat lifecycle (started / stopped — these are NOTIFICATIONS, not errors)
- **-988..-990**: node/process events
- **-1023..-1025**: replication delay
- **-1031..-1034**: apply failures
- **-1035**: applier OOM (`ha_log_applier_max_mem_size` exceeded)
- **-1036/1037**: applier/writer signal-shutdown
- **-1122/1133/1134**: copylogdb/applylogdb specific errors
- **-1139..-1144**: handshake errors

## `db_ha_apply_info` catalog
- One row per replicated DB tracking applier state (LSA, last applied time, last received time, last commit, last error).
- Timestamp must match log DB-creation time or applier rejects with "Failed to initialize db_ha_apply_info".

## Cross-References

- [[components/heartbeat]] — `cub_master` heartbeat protocol implementation
- [[components/log-manager]] — log records that get replicated
- [[components/recovery]] — applier replays log records into local DB
- [[components/cub-master]] · [[components/cub-master-main]] — master process orchestration
- [[sources/cubrid-manual-config-params]] — full HA parameter list
- [[sources/cubrid-manual-error-codes]] — `-898..-1170` HA error catalogue

## Incidental Wiki Enhancements

- [[components/heartbeat]]: documented `ha_log_applier_max_mem_size` protective shutdown semantics (error -1035, triggered when exceeded OR grown >2× quickly); `ha_delay_limit` paired -1025/-1026 events; `db_ha_apply_info` timestamp-must-match-log-creation rejection.
- [[components/cub-master]]: documented split-brain detection via `ha_ping_hosts` and the `[Diagnosis]/[Success]/[Canceled]` event prefixes in `<host>.cub_master.err`.
- [[components/recovery]]: documented HA `force_remove_log_archives=no` requirement to preserve archive logs needed by `applylogdb`.

## Key Insight

CUBRID HA is **shared-nothing log replication** with master-as-source-of-truth, slave-eligible-for-promotion, and replica-as-pure-read-scale. The major operational gotcha is that **failback is manual** — after a failover you must explicitly restart the old master as a slave to restore the original topology. The ha_mode parameter cannot be changed dynamically (restart required), and `force_remove_log_archives` MUST be `no` under HA. Everything else is tunable.
