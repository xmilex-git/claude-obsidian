---
created: 2026-04-27
type: source
title: "CUBRID Manual — Introduction (intro.rst)"
source_path: "/home/cubrid/cubrid-manual/en/intro.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - architecture
  - overview
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[Architecture Overview]]"
  - "[[components/cub-master]]"
  - "[[components/cub-server-main]]"
  - "[[components/broker-impl]]"
  - "[[components/cas]]"
  - "[[components/sp]]"
  - "[[components/double-write-buffer]]"
  - "[[Build Modes (SERVER SA CS)]]"
---

# CUBRID Manual — Introduction (intro.rst)

**Ingested:** 2026-04-27
**Source file:** `/home/cubrid/cubrid-manual/en/intro.rst` (284 lines)

## What This Chapter Covers

The high-level system tour: process structure, database volume layout, server/master/broker roles, interface modules, and a feature highlights pass (transactions, backup, partitioning, indexing, HA, stored procedures, click counter, OO extensions).

## Section Map

| Section | Content |
|---|---|
| **System Architecture → Process Structure** | The four logical components: database server, broker, CUBRID Manager, **PL server (`cub_pl`)** — note that `cub_pl` is now listed as a top-level component since 11.4 |
| **System Architecture → Database Volume Structure** | Permanent / Temporary / Backup volume taxonomy. Permanent: data, control file, active log, **DWB file**, **TDE key file**. Temporary: created on demand, identifier counts down from 32766. Archive log + background-archive log. |
| **System Architecture → Database Server** | One server process per DB. `max_clients` parameter. Master process handles connection setup. Two execution modes: client/server (CS) vs standalone (SA). |
| **System Architecture → Broker** | 3-layered broker: app → cub_broker → cub_cas → cub_server. Explicit note: cub_cas links to client lib, performs *parsing/optimization/plan creation* on the broker side. |
| **System Architecture → Interface Module** | JDBC / ODBC / OLE DB / PHP / CCI summary; ODBC and OLE DB are written on top of CCI. |
| **CUBRID Characteristics** | Bullet feature list: ACID, online/offline/incremental backup, range/hash/list partitioning, descending/covering/skip-order indexes, HA shared-nothing replication, stored procedures (PL/CSQL + Java), Click Counter (`INCR`/`WITH INCREMENT FOR`), collections (SET/MULTISET/LIST), JSON, inheritance. |

## Key Facts

- **Four CUBRID components** (since 11.4 explicitly enumerated): database server, broker, CUBRID Manager, **PL server (`cub_pl`)**. PL server is now listed as a peer of the database server, not a child concern.
- **GLO is gone**: existing apps using the GLO class must convert to BLOB/CLOB before migration.
- **Generic / data / index volume distinction is deprecated** — `cubrid createdb` still accepts the flags but treats all the same. Only "permanent data" vs "temporary data" matters.
- **Temporary volume identifier** counts **down** from 32766 (`<db>_t32766`, `_t32765`, …). Confirms the source-code observation in [[components/storage]].
- **temp_file_memory_size_in_pages**: in-memory temp limit before spilling to disk.
- **temp_file_max_size_in_pages**: caps temp on-disk usage (default `-1` = unlimited; `0` disables temp volume creation, forcing reliance on permanent volumes assigned for temporary use).
- **temp_volume_path**: override storage location of temp volumes (default = same as first DB volume).
- **DWB file** (`<db>_dwb`) is now a documented permanent file (was implementation detail in earlier versions).
- **TDE key file** is a permanent volume — file-based key store, up to 128 master keys, default at the DB volume location, overridable via `tde_keys_file_path`.
- **Backup volumes are partitioned** by `backup_volume_max_size_bytes` parameter.
- **One server process per DB** — `max_clients` parameter caps concurrent client connections per server.
- **One master process per host** (more precisely: per port number specified in `cubrid.conf`). The master listens on TCP, accepts client connection, then hands off the socket to the server.
- **Client/Server vs Standalone mode**: cannot coexist. Once a server process is running for DB `X`, no SA-mode utility can touch `X` simultaneously.
- **Click Counter** is documented as a first-class feature: `INCR()` function + `WITH INCREMENT FOR` SELECT clause, atomic and **outside the user transaction** — designed for page-view counters where SELECT-then-UPDATE would lock-contend.
- **Collections**: SET (no dups, sorted by storage), MULTISET (dups allowed, unordered), LIST (dups + insertion order preserved).
- **Inheritance** is an explicit "OO extension to the relational model" — see [[sources/cubrid-manual-sql-dml]] (`sql/oodb.rst`) for syntax. Largely deprecated in practice.

## Cross-References

This chapter is the entry point that the rest of the manual hangs off. Implementation-side counterparts:

- Process structure → [[Architecture Overview]] · [[components/cub-master]] · [[components/cub-master-main]] · [[components/cub-server-main]] · [[components/broker-impl]] · [[components/cas]]
- Volume structure → [[components/storage]] · [[components/double-write-buffer]] · [[components/page-buffer]] · [[components/log-manager]]
- Build modes → [[Build Modes (SERVER SA CS)]]
- PL server → [[components/sp]] · [[modules/pl_engine]] (if present)

## Incidental Wiki Enhancements

Applied during this ingest:

- [[Architecture Overview]]: confirmed PL server (`cub_pl`) is now a top-level documented component (4th tier alongside server / broker / CUBRID Manager) rather than an implementation detail.
- [[components/storage]] / [[components/double-write-buffer]]: DWB file naming `<db>_dwb` and TDE key file naming `<db>_keys` now have a documented manual citation (was implementation-only).

## Key Insight

The manual's intro chapter is the canonical "what is CUBRID" elevator pitch — read it before trying to onboard a non-CUBRID engineer. The vault's [[Architecture Overview]] is the implementation-flavoured counterpart; both should align.
