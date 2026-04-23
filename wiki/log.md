---
type: meta
title: "Operation Log"
updated: 2026-04-23
tags:
  - meta
  - log
status: evergreen
related:
  - "[[index]]"
  - "[[hot]]"
  - "[[overview]]"
  - "[[sources/_index]]"
---

# Operation Log

Navigation: [[index]] | [[hot]] | [[overview]]

Append-only. New entries go at the TOP. Never edit past entries.

Entry format: `## [YYYY-MM-DD] operation | Title`

Parse recent entries: `grep "^## \[" wiki/log.md | head -10`

---

## [2026-04-23] ingest | CUBRID round 3e — `api`, `debugging`, `win_tools`, `heaplayers` (parallel)
- Sources: `.raw/cubrid/src/{api,debugging,win_tools,heaplayers}/`
- Summaries: [[cubrid-src-api]], [[cubrid-src-debugging]], [[cubrid-src-win-tools]], [[cubrid-src-heaplayers]]
- Pages created (5): [[components/api|api]], [[components/cubrid-log-cdc|cubrid-log-cdc]], [[components/debugging|debugging]], [[components/win-tools|win-tools]], [[components/heaplayers|heaplayers]]
- Key insights: CDC API bypasses broker/CAS (raw CSS connection to cub_server); `strict_warnings` listed in AGENTS.md but absent from tree (gap noted); Windows NT service uses SCM control codes 160-223 + registry-key sync-by-convention; `src/heaplayers/` is unmodified Emery Berger vendor copy (Apache 2.0), engine surface = `HL_HEAPID` opaque handle only.

## [2026-04-23] ingest | CUBRID round 3d — `executables`, `monitor`, `session`, `cm_common` (parallel)
- Sources: `.raw/cubrid/src/{executables,monitor,session,cm_common}/`
- Summaries: [[cubrid-src-executables]], [[cubrid-src-monitor]], [[cubrid-src-session]], [[cubrid-src-cm-common]]
- Pages created (12): [[components/executables|executables]], [[components/cub-server-main|cub-server-main]], [[components/csql-shell|csql-shell]], [[components/cub-master-main|cub-master-main]], [[components/utility-binaries|utility-binaries]], [[components/monitor|monitor]], [[components/perfmon|perfmon]], [[components/stats-collection|stats-collection]], [[components/session|session]], [[components/session-state|session-state]], [[components/session-variables|session-variables]], [[components/cm-common-src|cm-common-src]]
- Key insights: csql runtime `dlopen()` DSO lets one binary serve SA+CS without recompile; cub_master single-threaded `select()` loop with C++ `master_server_monitor` for auto-respawn; monitor stats always-on (no sampling), per-tran sheets reused without zeroing → snapshot delta required; session zero-hash hot path via `thread_p->conn_entry->session_p` pointer cache; `@vars` & session params NOT rolled back; each session owns its own page-buffer LRU zone.

## [2026-04-23] ingest | CUBRID round 3c — `broker(impl)`, `communication`, `method`, `loaddb` (parallel)
- Sources: `.raw/cubrid/src/{broker,communication,method,loaddb}/`
- Summaries: [[cubrid-src-broker]], [[cubrid-src-communication]], [[cubrid-src-method]], [[cubrid-src-loaddb]]
- Pages created (14): [[components/broker-impl|broker-impl]], [[components/cas|cas]], [[components/broker-shm|broker-shm]], [[components/shard-broker|shard-broker]], [[components/communication|communication]], [[components/packer|packer]], [[components/request-response|request-response]], [[components/method|method]], [[components/method-invoke-group|method-invoke-group]], [[components/method-scan|method-scan]], [[components/loaddb|loaddb]], [[components/loaddb-grammar|loaddb-grammar]], [[components/loaddb-executor|loaddb-executor]], [[components/loaddb-driver|loaddb-driver]]
- Key insights: broker ↔ CAS are separate OS processes (only IPC = 2 POSIX shm + socket fd via SCM_RIGHTS); 44 dispatch codes in CAS; shard `CON_STATUS_LOCK` uses POSIX sem on Linux, Peterson's algorithm on Windows; no server-server RPC layer (HA replication uses ordinary client-facing slots); `cubpacking::packer` shared with XASL stream; `method_invoke_group` struct lives in `src/sp/` but instantiated by method scanner (sp+method inseparable); loaddb has no parse tree (streaming model: grammar action → callback → `locator_multi_insert_force`).

## [2026-04-23] ingest | CUBRID round 3b — `compat`, `sp`, `thread`, `connection` (parallel)
- Sources: `.raw/cubrid/src/{compat,sp,thread,connection}/`
- Summaries: [[cubrid-src-compat]], [[cubrid-src-sp]], [[cubrid-src-thread]], [[cubrid-src-connection]]
- Pages created (19): [[components/compat|compat]], [[components/db-value|db-value]], [[components/client-api|client-api]], [[components/dbi-compat|dbi-compat]], [[components/sp|sp]], [[components/sp-jni-bridge|sp-jni-bridge]], [[components/sp-method-dispatch|sp-method-dispatch]], [[components/sp-protocol|sp-protocol]], [[components/thread|thread]], [[components/thread-manager|thread-manager]], [[components/worker-pool|worker-pool]], [[components/entry-task|entry-task]], [[components/thread-daemon|thread-daemon]], [[components/connection|connection]], [[components/cub-master|cub-master]], [[components/network-protocol|network-protocol]], [[components/heartbeat|heartbeat]], [[components/tcp-layer|tcp-layer]]
- Key insights: `DB_VALUE` is 3-field struct; `DB_TYPE` enum ABI-frozen on disk + XASL stream (new types append after `DB_TYPE_JSON=40` only); cub_pl is **separate OS process** (no in-process JNI), Unix domain socket + bidirectional callback loop; cub_master uses `SCM_RIGHTS sendmsg` for zero-copy fd handoff (out of data path after handshake); local clients auto-upgrade to Unix domain socket (no TCP overhead).

## [2026-04-23] ingest | CUBRID round 3a — `transaction`, `object`, `base`, `xasl` (parallel)
- Sources: `.raw/cubrid/src/{transaction,object,base,xasl}/`
- Summaries: [[cubrid-src-transaction]], [[cubrid-src-object]], [[cubrid-src-base]], [[cubrid-src-xasl]]
- Pages created (24): [[components/mvcc|mvcc]], [[components/lock-manager|lock-manager]], [[components/deadlock-detection|deadlock-detection]], [[components/log-manager|log-manager]], [[components/recovery|recovery]], [[components/vacuum|vacuum]], [[components/server-boot|server-boot]], [[components/object|object]], [[components/schema-manager|schema-manager]], [[components/system-catalog|system-catalog]], [[components/authenticate|authenticate]], [[components/lob-locator|lob-locator]], [[components/base|base]], [[components/error-manager|error-manager]], [[components/memory-alloc|memory-alloc]], [[components/lockfree|lockfree]], [[components/system-parameter|system-parameter]], [[components/porting|porting]], [[components/xasl|xasl]], [[components/xasl-stream|xasl-stream]], [[components/regu-variable|regu-variable]], [[components/xasl-predicate|xasl-predicate]], [[components/xasl-aggregate|xasl-aggregate]], [[components/xasl-analytic|xasl-analytic]]
- Pages updated: [[components/transaction]] (stub→comprehensive)
- Key insights: `wait_for_graph.c` is dead code (`ENABLE_UNUSED_FUNCTION` guard) — actual deadlock detection in `lock_manager.c` (CONTRADICTS AGENTS.md claim); vacuum physically lives in `src/query/` not `src/transaction/`; `authenticate_context` is C++ class, legacy `au_*` macros are shims (grep traps); `memory_wrapper.hpp` last-include is architectural (glibc placement-new conflict avoidance); lock-free ABA solved via epoch-based retirement (`lockfree::tran::system`); XASL serializes pointers as byte offsets, 256-bucket visited-pointer hashtable, UNPACK_SCALE=3 = server pre-allocates 3× stream size.

---

## [2026-04-23] ingest | CUBRID src/storage/ — Storage Layer
- Source: `.raw/cubrid/src/storage/` (57 files, AGENTS.md present)
- Summary: [[cubrid-src-storage]]
- Pages created: [[components/page-buffer|page-buffer]], [[components/btree|btree]], [[components/heap-file|heap-file]], [[components/file-manager|file-manager]], [[components/double-write-buffer|double-write-buffer]], [[components/overflow-file|overflow-file]], [[components/extendible-hash|extendible-hash]], [[components/external-sort|external-sort]], [[components/external-storage|external-storage]]
- Pages updated: [[components/storage|storage]] (stub → comprehensive), [[components/_index|components/_index]]
- Key insight: 3-zone LRU buffer pool; DWB recovery precedes WAL redo; WAL-ordering enforced inside `pgbuf_flush_with_wal`; LOB delete uses `LOG_POSTPONE`; B-tree dispatch is parameterized by 18 `btree_op_purpose` values.

## [2026-04-23] ingest | CUBRID src/parser/ — SQL Parser
- Source: `.raw/cubrid/src/parser/` (39 files, AGENTS.md present)
- Summary: [[cubrid-src-parser]]
- Pages created: [[components/parse-tree|parse-tree]], [[components/name-resolution|name-resolution]], [[components/semantic-check|semantic-check]], [[components/xasl-generation|xasl-generation]], [[components/view-transform|view-transform]], [[components/parser-allocator|parser-allocator]], [[components/show-meta|show-meta]]
- Pages updated: [[components/parser|parser]] (stub → comprehensive), [[components/_index|components/_index]]
- Key insight: PT_NODE function tables ordinal-indexed (silent crash on misorder); `YYMAXDEPTH 1000000` + `container_2..11` Bison helpers; `parser_block_allocator::dealloc` is no-op (arena lifetime); `mq_translate` runs `mq_reset_ids` per view inline.

## [2026-04-23] ingest | CUBRID src/query/ — XASL Execution Layer
- Source: `.raw/cubrid/src/query/` (84 top-level files, AGENTS.md present; parallel/ excluded — separate ingest)
- Summary: [[cubrid-src-query]]
- Pages created: [[components/query|query]], [[components/query-executor|query-executor]], [[components/scan-manager|scan-manager]], [[components/cursor|cursor]], [[components/partition-pruning|partition-pruning]], [[components/dblink|dblink]], [[components/list-file|list-file]], [[components/aggregate-analytic|aggregate-analytic]], [[components/filter-pred-cache|filter-pred-cache]], [[components/memoize|memoize]]
- Pages updated: [[components/_index]], [[sources/_index]]
- Key insight: `qexec_execute_mainblock` ~27K lines (intentional); `SCAN_ID` polymorphic over 15 scan types incl. PARALLEL_HEAP_SCAN/DBLINK/JSON_TABLE/METHOD; hash GROUP BY 2-phase spill (2000 tuple calibration, 50% selectivity); memoize self-disables after 1000 iters at <50% hit rate; partition pruning enables O(1) MIN/MAX on partitioned tables; filter_pred_cache exclusive lease (no shared locks).

## [2026-04-23] ingest | CUBRID src/query/parallel/ — Parallel Query Execution
- Source: `.raw/cubrid/src/query/parallel/` (16 files + 3 subdirs)
- Summary: [[cubrid-src-query-parallel]]
- Pages created: [[components/parallel-query|parallel-query]], [[components/parallel-worker-manager|parallel-worker-manager]], [[components/parallel-task-queue|parallel-task-queue]], [[components/parallel-hash-join|parallel-hash-join]], [[components/parallel-heap-scan|parallel-heap-scan]], [[components/parallel-query-execute|parallel-query-execute]], [[components/parallel-sort|parallel-sort]]
- Key insight: single global named pool ("parallel-query") with lock-free CAS reservation; logarithmic auto-degree (`floor(log2(pages/threshold))+2`); thread-local errors must be moved to shared `err_messages_with_lock`; spin-yield wait (not condvar) for short-lived bursts; SERVER_MODE/SA_MODE only.

## [2026-04-23] ingest | CUBRID AGENTS.md (project guide)
- Source: `.raw/cubrid/AGENTS.md` (md5 946ec27...)
- Summary: [[cubrid-AGENTS]]
- Pages created: [[CUBRID]], [[Query Processing Pipeline]], [[Build Modes (SERVER SA CS)]], [[Memory Management Conventions]], [[Error Handling Convention]], [[Code Style Conventions]], [[modules/src|src]], [[modules/broker|broker]], [[modules/pl_engine|pl_engine]], [[modules/unit_tests|unit_tests]], [[components/parser]], [[components/optimizer]], [[components/storage]], [[components/transaction]]
- Pages updated: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Key Decisions]], [[index]]
- Key insight: same source tree compiles into 3 binaries (`SERVER_MODE`/`SA_MODE`/`CS_MODE`); parser+optimizer run client-side; `.c` files are compiled as C++17.

## [2026-04-23] scaffold | CUBRID Mode B overlay
- Type: scaffold
- Mode: B (GitHub / codebase)
- Source tree: /Users/song/DEV/cubrid
- Created folders: wiki/modules, wiki/components, wiki/decisions, wiki/dependencies, wiki/flows
- Created hub pages: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Dependency Graph]], [[Key Decisions]]
- Created _templates: module, component, decision, dependency, flow
- Updated CLAUDE.md with CUBRID scope and Mode B conventions

## [2026-04-08] save | claude-obsidian v1.4 Release Session
- Type: session
- Location: wiki/meta/claude-obsidian-v1.4-release-session.md
- From: full release cycle covering v1.1 (URL/vision/delta tracking, 3 new skills), v1.4.0 (audit response, multi-agent compat, Bases dashboard, em dash scrub, security history rewrite), and v1.4.1 (plugin install command hotfix)
- Key lessons: plugin install is 2-step (marketplace add then install), allowed-tools is not valid frontmatter, Bases uses filters/views/formulas not Dataview syntax, hook context does not survive compaction, git filter-repo needs 2 passes for full scrub

## [2026-04-08] ingest | Claude + Obsidian Ecosystem Research
- Type: research ingest
- Source: `.raw/claude-obsidian-ecosystem-research.md`
- Queries: 6 parallel web searches + 12 repo deep-reads
- Pages created: [[claude-obsidian-ecosystem]], [[cherry-picks]], [[claude-obsidian-ecosystem-research]], [[Ar9av-obsidian-wiki]], [[Nexus-claudesidian-mcp]], [[ballred-obsidian-claude-pkm]], [[rvk7895-llm-knowledge-bases]], [[kepano-obsidian-skills]], [[Claudian-YishenTu]]
- Key finding: 16+ active Claude+Obsidian projects; 13 cherry-pick features identified for v1.3.0+
- Top gap confirmed: no delta tracking, no URL ingestion, no auto-commit

## [2026-04-07] session | Full Audit, System Setup & Plugin Installation
- Type: session
- Location: wiki/meta/full-audit-and-system-setup-session.md
- From: 12-area repo audit, 3 fixes, plugin installed to local system, folder renamed

## [2026-04-07] session | claude-obsidian v1.2.0 Release Session
- Type: session
- Location: wiki/meta/claude-obsidian-v1.2.0-release-session.md
- From: full build session — v1.2.0 plan execution, cosmic-brain→claude-obsidian rename, legal/security audit, branded GIFs, PDF install guide, dual GitHub repos


- Source: `.raw/` (first ingest)
- Pages updated: [[index]], [[log]], [[hot]], [[overview]]
- Key insight: The wiki pattern turns ephemeral AI chat into compounding knowledge — one user dropped token usage by 95%.

## [2026-04-07] setup | Vault initialized

- Plugin: claude-obsidian v1.1.0
- Structure: seed files + first ingest complete
- Skills: wiki, wiki-ingest, wiki-query, wiki-lint, save, autoresearch
