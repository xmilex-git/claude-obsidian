---
created: 2026-04-23
type: meta
title: "Sources Index"
updated: 2026-04-27
tags:
  - meta
  - index
  - source
status: evergreen
related:
  - "[[index]]"
  - "[[log]]"
  - "[[entities/_index]]"
  - "[[CUBRID]]"
  - "[[sources/cubrid-manual-en-overview]]"
---

# Sources Index

Navigation: [[index]] | [[concepts/_index|Concepts]] | [[entities/_index|Entities]]

All source pages — summaries of ingested documents, transcripts, articles, and data.

---

## Transcripts

- [[sources/2026-04-28-tfile-role-analysis|2026-04-28-tfile-role-analysis]] — session note: role of `QMGR_TEMP_FILE *tfile` on the read/free path; decision-prep for integrating PR #6981's pattern into PR #7062's `px_scan_input_handler_list` (2026-04-28)

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
- [[sources/cubrid-src-heaplayers|cubrid-src-heaplayers]] — `src/heaplayers/` bundled Heap Layers (Emery Berger); `lea_heap.c` ~181 KB dlmalloc; 3rd-party, do-not-modify (2026-04-23)
- [[sources/cubrid-src-api|cubrid-src-api]] — `src/api/` CDC Change Data Capture client API: cubrid_log.h/c, four-phase model, LSA-based log extraction (2026-04-23)
- [[sources/cubrid-src-win-tools|cubrid-src-win-tools]] — `src/win_tools/` Windows NT service host, CLI control client, MFC tray app (2026-04-23)
- [[sources/cubrid-msg|cubrid-msg]] — `msg/` localized message catalogs: POSIX catgets format, three catalogs, four locales, gencat + iconv build pipeline (2026-04-23)
- [[sources/cubrid-3rdparty|cubrid-3rdparty]] — `3rdparty/` bundled/downloaded dependencies: 10 libraries, CMake ExternalProject orchestration (2026-04-23)
- [[sources/cubrid-AGENTS|cubrid-AGENTS]] — root `AGENTS.md` project guide: structure map, query pipeline, build modes, anti-patterns (2026-04-23)
- [[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]] — `src/query/parallel/` parallel execution: worker pool, hash join, heap scan, sort (2026-04-23)
- [[sources/cubrid-src-parser|cubrid-src-parser]] — `src/parser/` SQL parser, PT_NODE, Bison grammar, semantic check (2026-04-23)
- [[sources/cubrid-src-storage|cubrid-src-storage]] — `src/storage/` page buffer, B-tree, heap file, DWB, external storage (2026-04-23)
- [[sources/cubrid-src-cm-common|cubrid-src-cm-common]] — `src/cm_common/` CUBRID Manager shared utilities (2026-04-23)
- [[sources/cubrid-locales|cubrid-locales]] — `locales/` LDML data + build toolchain for per-locale shared libs (2026-04-23)
- [[sources/cubrid-timezones|cubrid-timezones]] — `timezones/` IANA tzdata + build toolchain → libcubrid_timezones (2026-04-23)
- [[sources/cubrid-src-query-operators|cubrid-src-query-operators]] — `src/query/` operator & evaluator family: arithmetic, numeric, string, regex, crypto, opfunc, evaluator (2026-04-23)
- [[sources/cubrid-contrib|cubrid-contrib]] — `contrib/` contributor drivers (Python/PHP/Perl/Ruby/.NET/Hibernate), observability, deployment (2026-04-23)
- [[sources/cubrid-query-scan-family|cubrid-query-scan-family]] — `src/query/` scan-family deep dive: hash, set, show, JSON_TABLE, partition pruning (2026-04-23)

---

## CUBRID 11.4 English Manual (catalog ingest, 2026-04-27)

End-user documentation companion to the source-code wiki. **Hub**: [[sources/cubrid-manual-en-overview|cubrid-manual-en-overview]] — overview of all 22 manual source pages with TOC.

- [[sources/cubrid-manual-en-overview|cubrid-manual-en-overview]] — manual hub + section TOC
- [[sources/cubrid-manual-intro|cubrid-manual-intro]] — `intro.rst`: architecture + features
- [[sources/cubrid-manual-install|cubrid-manual-install]] — install/upgrade/env/quickstart
- [[sources/cubrid-manual-csql|cubrid-manual-csql]] — CSQL shell + GUI tools
- [[sources/cubrid-manual-ha|cubrid-manual-ha]] — high availability (4194 lines)
- [[sources/cubrid-manual-shard|cubrid-manual-shard]] — database sharding middleware
- [[sources/cubrid-manual-security|cubrid-manual-security]] — TDE, SSL/TLS, ACL, authorization
- [[sources/cubrid-manual-admin|cubrid-manual-admin]] — admin guide: utilities, control, scripts, troubleshoot, systemtap
- [[sources/cubrid-manual-config-params|cubrid-manual-config-params]] — system parameter reference (config.rst)
- [[sources/cubrid-manual-error-codes|cubrid-manual-error-codes]] — 792-code error catalogue
- [[sources/cubrid-manual-api|cubrid-manual-api]] — drivers overview (PHP, PDO, ODBC, ADO.NET, Perl, Python, Ruby, Node.js)
- [[sources/cubrid-manual-cci|cubrid-manual-cci]] — CCI C API (95 functions)
- [[sources/cubrid-manual-jdbc|cubrid-manual-jdbc]] — JDBC driver reference
- [[sources/cubrid-manual-pl|cubrid-manual-pl]] — stored procedures (Java SP, methods, cross-cutting)
- [[sources/cubrid-manual-plcsql|cubrid-manual-plcsql]] — PL/CSQL language reference (NEW 11.4)
- [[sources/cubrid-manual-sql-foundation|cubrid-manual-sql-foundation]] — types, literals, i18n, transactions, auth, catalog
- [[sources/cubrid-manual-sql-dml|cubrid-manual-sql-dml]] — queries, DBLink, OODB
- [[sources/cubrid-manual-sql-ddl|cubrid-manual-sql-ddl]] — schema, trigger, partition
- [[sources/cubrid-manual-sql-tuning-parallel|cubrid-manual-sql-tuning-parallel]] — optimizer hints, parallel execution
- [[sources/cubrid-manual-sql-functions|cubrid-manual-sql-functions]] — operators + function library
- [[sources/cubrid-manual-release-notes-114|cubrid-manual-release-notes-114]] — CUBRID 11.4 release notes

---

## Articles

<!-- Add article source pages here -->

---

## Papers

<!-- Add paper source pages here -->

---

## Add new sources here after each ingest.
