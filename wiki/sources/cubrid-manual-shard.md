---
created: 2026-04-27
type: source
title: "CUBRID Manual — Database Sharding (shard.rst)"
source_path: "/home/cubrid/cubrid-manual/en/shard.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - shard
  - middleware
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[components/shard-broker]]"
  - "[[components/broker-impl]]"
  - "[[sources/cubrid-manual-config-params]]"
---

# CUBRID Manual — Database Sharding (shard.rst)

**Ingested:** 2026-04-27
**Source file:** `/home/cubrid/cubrid-manual/en/shard.rst` (957 lines)

## What This Is

CUBRID SHARD is **middleware** that sits between applications and a set of horizontally-partitioned backend DBs. It is a **separate process tree** (broker + proxy + CAS), distinct from the regular cub_broker. Routing decisions are made via SQL hints + a configuration file mapping shard keys to backend DBs.

## Section Map

| Section | Content |
|---|---|
| **Database Sharding** | Conceptual intro; horizontal partitioning |
| **CUBRID SHARD Terminologies** | shard, shard_key, shard_id, shard DB, proxy, etc. |
| **Main Features** | Middleware structure (broker / proxy / CAS), shard SQL hint selection, transaction support |
| **Quick Start** | 4-shard config example: cubrid_broker.conf, shard_connection.txt, shard_key.txt; sample Java program |
| **Architecture and Configuration** | broker-proxy-CAS architecture, default config, shard_connection.txt format, shard_key.txt format |
| **Shard SQL Hints** | `/*+ shard_key(...) */`, `/*+ shard_val(...) */`, `/*+ shard_id(...) */` |
| **Operations** | `cubrid broker start --shard`, `broker_status`, shard-specific monitoring |
| **Limitations** | What SHARD does NOT support (cross-shard joins, distributed txn 2PC, etc.) |

## Key Facts

### Architecture
- **Three-process middleware**: shard_broker → shard_proxy → shard_cas. Each plays a distinct role:
  - `shard_broker` — accepts client connections, like a normal broker
  - `shard_proxy` — routes by shard key, manages backend connections
  - `shard_cas` — terminating CAS that talks to the actual backend shard DB
- Backend shard DB can be any CUBRID instance (manual implies CUBRID-only; cross-engine via DBLink is separate).
- **Up to 256 shards** per CUBRID SHARD configuration.

### Shard key & routing
- **Shard key column** — a column in the schema chosen as the partition key (e.g., `student_no`).
- **Shard mapping** is a hash — `shard_key.txt` defines `<hash range> → <shard ID>` mappings. Default modular hash; library and function hash also supported.
- **Backend connections** defined in `shard_connection.txt` — one line per shard ID with hostname/port/db/user/pw.

### SQL Hints (the routing primitives)
- **`/*+ shard_key(<col>) */`** — placed before bind variable or literal in WHERE clause; routes to shard determined by that value.
- **`/*+ shard_val(<value>) */`** — explicit value override when no shard column appears in the query.
- **`/*+ shard_id(<id>) */`** — bypass hash; explicit shard selection by ID.
- Without a hint or a routable predicate, the query is **broadcast** to all shards (or rejected — depends on config).

### Configuration files
- **`cubrid_broker.conf`** — refer to `cubrid_broker.conf.shard` template for shard-specific parameters.
- **`shard_connection.txt`** (`SHARD_CONNECTION_FILE`) — backend DB list. Format: `<shard_id> <db_name> <host>` (port from `cubrid_port_id` of the backend's `cubrid.conf`, NOT in this file for CUBRID).
- **`shard_key.txt`** (`SHARD_KEY_FILE`) — hash range → shard ID mapping.

### Transaction support
- ACID guaranteed **per backend shard DB** (because each shard is just a regular CUBRID DB).
- **No 2PC** across shards — cross-shard transactions are not atomic. The middleware sends rollback to the backing shard if the application aborts, but multi-shard atomicity is not guaranteed.

### Operational
- **Shard broker uses dedicated `[%shard_*]` blocks** in `cubrid_broker.conf`.
- `cubrid_port_id` in backend `cubrid.conf` must match what the shard middleware tries to connect to.
- Standard broker ops (`cubrid broker start/stop/status`) apply, with `--shard` flag for the shard-specific binary.

### Notable shard params (from config.rst)
- `SHARD_KEY_MODULAR` — modular value for the default modular hash.
- `SHARD_KEY_LIBRARY_NAME` / `SHARD_KEY_FUNCTION_NAME` — custom hash function loaded from a `.so`.
- `SHARD_NUM_PROXY` — proxy worker count.
- `SHARD_MAX_CLIENTS` — default 256 client connections per proxy.
- `SHARD_MAX_PREPARED_STMT_COUNT` — default 10,000.
- `SHARD_PROXY_TIMEOUT` — default 30 s.

## Limitations
- **No cross-shard joins** at the middleware layer.
- **No distributed transactions** (no 2PC).
- **No automatic resharding** — adding/removing shards requires re-bucketing data manually (export from old, import to new shard).
- Sharding is orthogonal to HA: each backend shard DB can independently be HA-protected, but the middleware itself is not HA.

## Cross-References

- [[components/shard-broker]] — `src/broker/shard_*` implementation
- [[components/broker-impl]] — non-shard broker comparison
- [[sources/cubrid-manual-config-params]] — `SHARD_*` parameter reference
- [[components/dblink]] · [[sources/cubrid-manual-sql-dml]] (`sql/dblink.rst`) — alternative for cross-DB queries (single-query, not a sharding solution)

## Incidental Wiki Enhancements

- [[components/shard-broker]]: documented the 256-shard cap, the broker/proxy/CAS three-process architecture, the three SQL hint forms (`shard_key`/`shard_val`/`shard_id`), and the `cubrid_port_id`-from-backend-cubrid.conf indirection for CUBRID backend port.

## Key Insight

CUBRID SHARD is a **router**, not a distributed transaction manager. It scales reads and writes that respect a single shard key, but cross-shard atomicity is the application's problem. For most "scale out" use cases that don't need shard-aware logic, **HA replicas** for read scaling + **partitioning within a single DB** for write distribution are simpler than SHARD. SHARD is for genuine multi-DB horizontal scale where the application can route by key.
