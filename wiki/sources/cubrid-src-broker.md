---
type: source
title: "CUBRID src/broker/ — Connection Broker & CAS"
source_path: "src/broker/"
date_ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - broker
  - cas
  - shared-memory
  - multi-process
  - sharding
related:
  - "[[components/broker-impl|broker-impl]]"
  - "[[components/cas|cas]]"
  - "[[components/broker-shm|broker-shm]]"
  - "[[components/shard-broker|shard-broker]]"
  - "[[modules/broker|broker]]"
---

# Source: `src/broker/`

**Ingested:** 2026-04-23
**Coverage:** `AGENTS.md`, `broker.c`, `cas.c`, `broker_shm.h`, `shard_proxy.c`, `shard_metadata.c`, `cas_execute.c` (headers), directory listing via AGENTS.md

## What This Directory Is

`src/broker/` implements the **middle tier** of CUBRID's three-tier client-server architecture. It is entirely separate from the top-level `broker/` CMake target directory — that directory is just build scaffolding; this is the actual code.

Two primary process types live here:

1. **Broker process** (`broker.c`) — single multi-threaded process per broker config entry; owns the TCP listener socket, dispatches incoming connections to CAS via shared memory + fd-passing.
2. **CAS process** (`cas.c`) — one OS process per concurrent client session; receives client fd, opens its own CSS session with `cub_server`, dispatches 44+ protocol function codes.

Optional **shard proxy** processes (`shard_proxy.c`) sit between broker and CAS when `SHARD=ON`.

## Key Files

| File | Role |
|------|------|
| `broker.c` | Broker main loop, receiver/dispatch/monitor threads, CAS lifecycle |
| `cas.c` | CAS entry point, `server_fn_table[]`, `cas_main()`, `shard_cas_main()` |
| `cas_common_main.c` | `cas_main_loop()` shared by normal and shard CAS |
| `cas_execute.c` | SQL execution: `fn_prepare`, `fn_execute`, `fn_fetch` |
| `cas_function.c` | Non-SQL protocol handlers |
| `cas_network.c/h` + `cas_net_buf.c/h` | CAS I/O and output buffering |
| `cas_handle.c` | Statement handle (`T_SRV_HANDLE`) lifecycle |
| `broker_shm.c/h` | Shared memory creation/attachment; semaphore wrappers |
| `broker_config.c` | `cubrid_broker.conf` parsing → `T_BROKER_INFO` |
| `broker_monitor.c` | `broker_monitor` CLI utility |
| `broker_admin.c` | `cubrid broker` admin commands |
| `broker_acl.c` | IP + dbuser access control |
| `broker_log_util.c` + `broker_log_top.c` | SQL/slow log analysis tools |
| `shard_proxy.c` | Shard proxy process lifecycle |
| `shard_proxy_handler.c` | Shard request routing |
| `shard_metadata.c` | Loads shard_user / shard_range / shard_conn tables |
| `shard_key_func.c` | MODULAR key fn + dlopen for custom routing |
| `shard_shm.c/h` | Shard-specific shm helpers |
| `shard_statement.c` | Prepared statement cache across shard CAS pool |

## Key Data Structures

| Struct | Purpose |
|--------|---------|
| `T_SHM_BROKER` | Global broker registry (all brokers' config) |
| `T_BROKER_INFO` | Per-broker config: port, min/max CAS, timeouts |
| `T_SHM_APPL_SERVER` | Per-broker CAS pool: job queue + all `as_info[]` slots |
| `T_APPL_SERVER_INFO` | Per-CAS status: uts_status, con_status, pid, counters |
| `T_SHM_PROXY` | Shard proxy configuration + all shard metadata |
| `T_PROXY_INFO` | Per-proxy-process status and statistics |
| `T_SHARD_KEY` / `T_SHARD_KEY_RANGE` | Range-based routing table |
| `T_SHARD_CONN` | shard_id → (db_name, db_host) |

## Architectural Insights

**Process isolation by design.** Broker and CAS are separate OS processes — this gives crash isolation (a buggy CAS dies without taking the broker), but requires careful synchronization via `CON_STATUS_LOCK` / `CON_STATUS_UNLOCK` around every `uts_status` / `con_status` update.

**Shared memory is the IPC bus.** The job queue (`T_MAX_HEAP_NODE job_queue[4097]`), all CAS status (`T_APPL_SERVER_INFO as_info[4096]`), and all configuration live in shared memory. This allows the `broker_monitor` utility to display live statistics with zero overhead on the broker process.

**Process-level connection pooling.** CAS processes are reused across sessions (`keep_connection=ON/AUTO`), making CUBRID's "connection pooling" fundamentally process-based rather than thread-based. This trades higher per-idle memory usage for simplicity and crash isolation.

**CAS function dispatch is a static table.** `server_fn_table[]` in `cas.c` is indexed directly by `CAS_FC_*` enum values — O(1) dispatch with no string comparison.

**Shard proxy is a whole extra tier.** The shard code (`shard_*.c`) is substantial — it adds async I/O, statement caching, key extraction from SQL, and a separate dlopen-based routing API. It is activated only when `SHARD=ON` in config.

## Pages Created

- [[components/broker-impl|broker-impl]] — component hub
- [[components/cas|cas]] — CAS worker process
- [[components/broker-shm|broker-shm]] — shared memory IPC
- [[components/shard-broker|shard-broker]] — shard proxy variant
