---
created: 2026-04-27
type: source
title: "CUBRID Manual — System Parameters (admin/config.rst)"
source_path: "/home/cubrid/cubrid-manual/en/admin/config.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - config
  - parameters
  - reference
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-admin]]"
  - "[[components/system-parameter]]"
  - "[[components/broker-shm]]"
  - "[[components/broker-impl]]"
---

# CUBRID Manual — System Parameters (config.rst)

**Ingested:** 2026-04-27
**Source file:** `/home/cubrid/cubrid-manual/en/admin/config.rst` (3304 lines)

## What This Is

The **definitive reference for every system parameter** in `cubrid.conf`, `cubrid_broker.conf`, and `cm.conf`. Hundreds of parameters, organized by functional group. Tabular format: parameter name, type, default, range, description.

This page is **deliberately a catalog only** — actual parameter lookups should hit the RST file directly via `grep` or section anchor, not be reproduced here.

## How to Look Up a Parameter

```bash
grep -n "^\.\. _<param_name>:" /home/cubrid/cubrid-manual/en/admin/config.rst
```
or by name:
```bash
grep -n -B 1 "<param_name>" /home/cubrid/cubrid-manual/en/admin/config.rst
```

## Section Map

| Section | Lines (approx) | Content |
|---|---|---|
| **Scope** | 25-86 | Client / server / client-server scopes; precedence rules |
| **Change channels** | 100-122 | File edit (cubrid.conf), `SET SYSTEM PARAMETERS '...=...'` SQL, `;set name=value` CSQL session command, DBA vs non-DBA scope, `DEFAULT` keyword reset (exception: `call_stack_dump_activation_list`) |
| **Connection** | ~535-700 | `cubrid_port_id` (1523), `max_clients` (default 100, max 4000), `check_peer_alive` (yes/server/client/both/none) |
| **Memory** | ~685-750 | `data_buffer_size` (32,768 × page_size default), `sort_buffer_size` (cap 2 GB), `temp_file_memory_size_in_pages`, `temp_file_max_size_in_pages` (-1 unlimited, 0 disabled) |
| **Disk / volumes** | ~750-900 | `db_volume_size`, `log_volume_size`, `db_page_size`, `log_page_size` (16K default), `temp_volume_path`, `backup_volume_max_size_bytes`, `db_block_size` |
| **Error logging** | ~900-1000 | `error_log_level` (NOTIFICATION default), `error_log_size` |
| **Locking** | 1110-1180 | `deadlock_detection_interval_in_secs` (1.0), `lock_escalation` (100,000), `rollback_on_lock_escalation`, `lock_timeout` (msec, -1=infinite, 0=nowait) |
| **MVCC isolation** | 1118-1142 | `isolation_level` (4=READ COMMITTED default, 5=REP READ, 6=SERIALIZABLE) |
| **Logging / WAL / checkpoint** | 1215-1290 | `log_buffer_size` (min 2 MB), `checkpoint_interval` (6 min), `checkpoint_every_size` (~156 MB), `force_remove_log_archives` (HA: must be `no`), `log_max_archives` (INT_MAX), `background_archiving` (yes default), `dwb_logging`, `double_write_buffer_size` (2 MB; 0 disables) |
| **Statement / query** | ~1300-1400 | `optimization_level`, `query_trace`, `sql_trace_slow`, `sql_trace_ioread_pages`, `query_cache_mode`, `subquery_cache` |
| **Thread / pool** | 1850-1900 | `thread_stacksize`, `max_parallel_workers` (100), `parallelism` (4) |
| **Timezone** | ~1900-1950 | `timezone`, `tz_leap_second_support` |
| **Plan / query cache** | ~2000-2100 | `xasl_cache_max_clones`, `subquery_cache_size_in_pages`, `query_cache_max_size_in_kb` |
| **Other** | 2300-2400 | `recovery_progress_logging_interval`, `vacuum_*`, `auto_restart_server` (NEW 11.4), `flashback_timeout` (300 s) |
| **HA params** | 2249-2257 | `ha_mode` (off/on/replica), no-dynamic-change |
| **PL server** | 2388-2394 | `stored_procedure` (yes default), `stored_procedure_port` (0=random), `stored_procedure_uds` (Linux only), `pl_transaction_control` (no default) |
| **Broker common params** | 2632-2780 | `MASTER_SHM_ID` (30001 default), `ADMIN_LOG_FILE`, `ACCESS_*` |
| **Broker per-broker `[%name]`** | 2746-2900 | `BROKER_PORT`, `MIN_NUM_APPL_SERVER` (5), `MAX_NUM_APPL_SERVER` (40), `APPL_SERVER_PORT`, `APPL_SERVER_SHM_ID`, `APPL_SERVER_MAX_SIZE_HARD_LIMIT` (1024 MB min, 2,097,151 max), `KEEP_CONNECTION` (ON/AUTO), `RECONNECT_TIME` (600 s), `TIME_TO_KILL` (120 s), `REPLICA_ONLY`, `PREFERRED_HOSTS`, `CONNECT_ORDER`, `MAX_NUM_DELAYED_HOSTS_LOOKUP`, `SQL_LOG_MAX_SIZE`, `SLOW_LOG_MAX_SIZE`, `SSL=ON|OFF`, `MAX_PREPARED_STMT_COUNT` (2000), `NET_BUF_SIZE` (NEW 11.4: 16K/32K/48K/64K), `ACCESS_CONTROL`, `ACCESS_CONTROL_FILE`, `ACCESS_CONTROL_DEFAULT_POLICY` (NEW 11.4: DENY/ALLOW) |
| **Broker SHARD params** | ~2900-3000 | `SHARD_KEY_MODULAR`, `SHARD_KEY_LIBRARY_NAME`, `SHARD_KEY_FUNCTION_NAME`, `SHARD_NUM_PROXY`, `SHARD_MAX_CLIENTS` (256), `SHARD_MAX_PREPARED_STMT_COUNT` (10,000), `SHARD_PROXY_TIMEOUT` (30 s), `SHARD_CONNECTION_FILE`, `SHARD_KEY_FILE` |
| **CM (manager) params** | ~3000-3300 | `cm_port` (8001), `cm_process_monitor_interval`, `cm_acl_file` |

## Key Defaults Worth Memorizing

### Connectivity
- `cubrid_port_id` = **1523** (master)
- `BROKER_PORT` = depends on broker (`query_editor` 30000, `broker1` 33000)
- `APPL_SERVER_SHM_ID` per broker (30000 query_editor, 33000 broker1)
- `MASTER_SHM_ID` = **30001**
- `cm_port` = **8001**

### Capacity / sizing
- `db_volume_size` = **512 MB** default
- `log_volume_size` = **512 MB** default
- `db_page_size` = **16 KB** default
- `data_buffer_size` = 32,768 × `db_page_size` = **512 MB** at default
- `max_clients` = 100 (max 4000)
- `MAX_NUM_APPL_SERVER` = 40 per broker, `MIN_NUM_APPL_SERVER` = 5

### Recovery / WAL
- `checkpoint_interval` = **6 min**
- `checkpoint_every_size` = 10,000 × log_page_size ≈ **156 MB**
- `log_buffer_size` minimum = **2 MB** (since 9.x; was 48 KB pre-9.x — old configs must increase)
- `double_write_buffer_size` = **2 MB** (0 disables DWB entirely)

### Locking / MVCC
- `deadlock_detection_interval_in_secs` = **1.0 s** (min 0.1)
- `lock_escalation` = **100,000**
- `lock_timeout` = -1 (infinite) by default
- `isolation_level` = **4 = READ COMMITTED**

### Parallel
- `parallelism` = **4** (capped at MIN(32, system_cores))
- `max_parallel_workers` = **100**
- `max_subquery_cache_size` = **2 MB**
- `max_hash_list_scan_size` = **8 MB**
- `max_agg_hash_size` = **2 MB**

### Session / app
- `session_state_timeout` = **21,600 s = 6 hours** (PREPARE, @vars, LAST_INSERT_ID, ROW_COUNT lifetime)
- `MAX_PREPARED_STMT_COUNT` (broker) = **2000**
- SQL-level PREPARE cap = **20** per DB connection (server protection)
- User-defined session vars cap = **20** per connection

### Vacuum
- `vacuum_ovfp_check_threshold` = **1000**
- `vacuum_ovfp_check_duration` = **45,000**
- `vacuum_prefetch_log_buffer_size` = **50 MB**

### TDE / encryption
- `tde_default_algorithm` = **AES** (alternative ARIA)
- `tde_keys_file_path` (default = data volume location)

### Other
- `error_log_level` = **NOTIFICATION** (most verbose: ERROR < NOTIFICATION < WARNING)
- `auto_restart_server` (NEW 11.4) = controls automatic cub_server restart in non-HA after abnormal termination
- `flashback_timeout` = **300 s** (mandatory minimum, intentionally not removable to prevent log retention buildup)
- `recovery_progress_logging_interval` controls log-recovery `-1128/-1129` cadence

## Change Channels (definitive)

1. **Edit `cubrid.conf` / `cubrid_broker.conf`** — restart required.
2. **`SET SYSTEM PARAMETERS '<name>=<v>; <name>=<v>'`** — SQL statement; effective immediately. DBA can change server params; non-DBA only session params.
3. **CSQL `;set <name>=<value>`** — session command; same scope as #2 from CSQL.
4. **`cubrid <util>` with command-line flags** — utility-specific overrides; ephemeral to that invocation.

`DEFAULT` keyword resets a param to its default (exception: `call_stack_dump_activation_list` does not honor `DEFAULT`).

## Scope (`config.rst:25-86`)

- **client**: client-side params (driver behavior, e.g., `cci_default_autocommit`)
- **server**: server-side params (engine internals)
- **client-server**: parameters that must match on both sides (e.g., `db_page_size`)

## Cross-References

- [[components/system-parameter]] — implementation of param storage, change channels, scope enforcement
- [[components/broker-shm]] — `MASTER_SHM_ID`, `APPL_SERVER_SHM_ID`
- [[components/broker-impl]] — `MAX_NUM_APPL_SERVER`, `KEEP_CONNECTION`, etc.
- [[sources/cubrid-manual-admin]] — `cubrid paramdump` for current effective values

## Incidental Wiki Enhancements

- [[components/system-parameter]]: documented the 4-channel change model (file edit, SQL `SET SYSTEM PARAMETERS`, CSQL `;set`, utility flag) and `DEFAULT` keyword reset behavior + `call_stack_dump_activation_list` exception.
- [[components/broker-shm]]: documented `MASTER_SHM_ID=30001` default, `APPL_SERVER_SHM_ID` per-broker (30000 query_editor, 33000 broker1), system-wide uniqueness requirement.
- [[components/broker-impl]]: documented `KEEP_CONNECTION=AUTO` switching policy (between connection-unit and transaction-unit binding based on CAS-vs-client count), `RECONNECT_TIME` 600 s default for failover-back to PREFERRED_HOSTS, `APPL_SERVER_MAX_SIZE_HARD_LIMIT` 1024-MB min and 2,097,151-MB max bounds, and the new 11.4 `NET_BUF_SIZE` (16K/32K/48K/64K) and `ACCESS_CONTROL_DEFAULT_POLICY` (DENY/ALLOW) parameters.

## Key Insight

`config.rst` is a **lookup-only reference** — too dense to summarize, too detailed to memorize. The handful of facts above (defaults, change channels, scope) cover 90% of what comes up in operations. For everything else, read the file directly.
