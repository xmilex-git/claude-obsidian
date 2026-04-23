---
type: meta
title: "Sources Index"
updated: 2026-04-07
tags:
  - meta
  - index
  - source
status: evergreen
related:
  - "[[index]]"
  - "[[log]]"
  - "[[entities/_index]]"
  - "[[Andrej Karpathy]]"
---

# Sources Index

Navigation: [[index]] | [[concepts/_index|Concepts]] | [[entities/_index|Entities]]

All source pages — summaries of ingested documents, transcripts, articles, and data.

---

## Transcripts


---

## CUBRID Codebase Sources

- [[sources/cubrid-src-compat|cubrid-src-compat]] — `src/compat/` client API & DB_VALUE (2026-04-23)
- [[sources/cubrid-src-sp|cubrid-src-sp]] — `src/sp/` stored procedure JNI bridge (2026-04-23)
- [[sources/cubrid-src-query|cubrid-src-query]] — `src/query/` XASL execution layer (2026-04-23)
- [[sources/cubrid-src-object|cubrid-src-object]] — `src/object/` schema, auth, catalog, LOB locator (2026-04-23)
- [[sources/cubrid-src-base|cubrid-src-base]] — `src/base/` core utilities: error, memory, lock-free, porting, system params (2026-04-23)
- [[sources/cubrid-src-xasl|cubrid-src-xasl]] — `src/xasl/` XASL node type definitions + `src/query/xasl.h` + `regu_var.hpp` (2026-04-23)
- [[sources/cubrid-src-transaction|cubrid-src-transaction]] — `src/transaction/` MVCC, WAL, locking, recovery, boot (2026-04-23)
- [[sources/cubrid-src-communication|cubrid-src-communication]] — `src/communication/` NET_SERVER_REQUEST_LIST dispatch, packer/unpacker, method callbacks, histogram (2026-04-23)
- [[sources/cubrid-src-connection|cubrid-src-connection]] — `src/connection/` CSS protocol, TCP, cub_master, HA heartbeat (2026-04-23)
- [[sources/cubrid-src-thread|cubrid-src-thread]] — `src/thread/` worker pools, daemons, THREAD_ENTRY (2026-04-23)
- [[sources/cubrid-src-method|cubrid-src-method]] — `src/method/` method/SP invocation from queries; S_METHOD_SCAN (2026-04-23)
- [[sources/cubrid-src-broker|cubrid-src-broker]] — `src/broker/` connection broker, CAS workers, shared memory IPC, shard proxy (2026-04-23)
- [[sources/cubrid-src-loaddb|cubrid-src-loaddb]] — `src/loaddb/` bulk loader: bison/flex grammar, parallel workers, direct heap insert (2026-04-23)
- [[sources/cubrid-src-monitor|cubrid-src-monitor]] — `src/monitor/` performance statistics: primitives, transaction sheets, global registry, VACUUM ovfp threshold (2026-04-23)
- [[sources/cubrid-src-session|cubrid-src-session]] — `src/session/` per-connection session state: SESSION_STATE struct, @var bindings, prepared statement cache, holdable cursors (2026-04-23)
- [[sources/cubrid-src-executables|cubrid-src-executables]] — `src/executables/` binary entry points: cub_server, csql, cub_master, admin utilities (2026-04-23)
- [[sources/cubrid-src-debugging|cubrid-src-debugging]] — `src/debugging/` compile-time type name helpers; header-only; zero release-build footprint (2026-04-23)

---

## Articles

<!-- Add article source pages here -->

---

## Papers

<!-- Add paper source pages here -->

---

## Add new sources here after each ingest.
