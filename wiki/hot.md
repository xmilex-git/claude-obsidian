---
created: 2026-04-23
type: meta
title: "Hot Cache"
updated: 2026-04-24
tags:
  - meta
  - hot
status: active
---

# Recent Context

## CUBRID Baseline Commit
**`175442fc858bd0075165729756745be6f8928036`** — all wiki claims, file paths, line numbers reflect this commit of `~/dev/cubrid/`. Before any new ingest, check repo HEAD; if newer, compute `git log <baseline>..HEAD -- <path>` and reconcile affected wiki pages before writing. Protocol: `CLAUDE.md` § "CUBRID Baseline Commit".

## PR Ingest
**On-demand only, user-specified only.** Do not autonomously scan, poll, or batch. All PR states are ingestable — behavior differs: merged-case-c → reconcile + bump now; merged-case-a/b → retroactive doc only; open/draft → write Reconciliation Plan (do NOT edit component pages for PR-induced changes); closed-unmerged → doc as abandoned. **Deep code analysis required** every time (read baseline source files, not just the diff). **Incidental Knowledge Enhancement expected** every time: facts about baseline code that are missing/wrong/incomplete in the wiki get added to component/source pages immediately, regardless of PR state. Deferred plan execution via "apply reconciliation for PR #NNNN". Template: `_templates/pr.md`; index: [[prs/_index|PRs]]; protocol: `CLAUDE.md` § "PR Ingest (user-specified only, all states accepted, code analysis required)".

## Last Updated
2026-04-27. **Manual ingest session.** Cataloged the full CUBRID 11.4 English User Manual (`/home/cubrid/cubrid-manual/en/` — 119 RST files, ~88K lines, 37 MB) into 22 `cubrid-manual-*` source pages. Strategy: catalog + enhance (NOT per-file ingest — 1-line summary per RST file, key facts inline, cross-refs back to RST tree). Hub: [[sources/cubrid-manual-en-overview]]. **4 parallel-agent dispatch** for sql/, admin/, api/, pl/ clusters. **Top-7 incidental enhancements applied** to component pages: [[components/cub-master]] (auto_restart_server NEW 11.4), [[components/sp]] (pl_transaction_control, AUTHID, recursion 16-via-query), [[components/lock-manager]] (BU_LOCK introduced 10.2), [[components/btree]] (online index 3-stage protocol), [[components/optimizer]] (28-hint catalog), [[components/error-manager]] (error namespace partitioning), [[components/recovery]] (-1128/-1129 NOTIFICATIONs + parallel REDO 11.4). ~40+ remaining enhancements documented in source pages, deferred. Manifest tree-hash: `85c1ebbc75933acda015a8776506a270`.

**Prior session (2026-04-26):** PR #7011 deep ingest (parallel index build, OPEN, 9 files). [[prs/PR-7011-parallel-index-build]] with full Reconciliation Plan. Resolved baseline gap re: `sort_copy_sort_param` location.

**Prior session (2026-04-24):** Lint + legacy cleanup. Filed `lint-report-2026-04-24`. 18 pre-CUBRID pages moved into `wiki/_legacy/`. Hub pages rewritten to strip legacy-first-class listings.

**Prior session (2026-04-23):** CUBRID deep-dive rounds 1–5 finished. 150 component pages, 34 source summaries, 246 total wiki md.

## Wiki shape
- `wiki/components/` (111 pages) — one section per CUBRID subsystem
- `wiki/sources/` (27 pages) — 21 are CUBRID source-tree ingests
- `wiki/modules/`, `wiki/decisions/`, `wiki/dependencies/`, `wiki/flows/` — Mode B scaffold (decisions/dependencies/flows still mostly empty — opportunity)
- Hub pages: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Dependency Graph]], [[Key Decisions]]
- Concept pages: [[Query Processing Pipeline]], [[Build Modes (SERVER SA CS)]], [[Memory Management Conventions]], [[Error Handling Convention]], [[Code Style Conventions]]

## Top-of-mind facts (most-cited across pages)

### Build / topology
- Same source compiles to 3 binaries via `SERVER_MODE` / `SA_MODE` / `CS_MODE` ([[Build Modes (SERVER SA CS)]]).
- 3-tier topology: client (CCI/JDBC) → broker → CAS workers → DB server. broker ↔ CAS = separate OS processes, only IPC = 2 POSIX shm + socket fd via `SCM_RIGHTS`.
- cub_master = single-threaded `select()` loop with C++ `master_server_monitor` for auto-respawn.
- Local clients auto-upgrade TCP → Unix domain socket.
- cub_pl (Java PL) = separate OS process (no in-process JNI).

### Query path
- Parser + optimizer **client-side** (`#if !defined(SERVER_MODE)`). Server only sees [[components/xasl|XASL]] byte stream.
- XASL serialization: pointers as byte offsets, 256-bucket visited-ptr hashtable, UNPACK_SCALE=3.
- `qexec_execute_mainblock` ~27 K lines = single dispatch for SELECT, all DML, set ops, CONNECT BY, MERGE.
- `SCAN_ID` = polymorphic union over **15** scan types.
- Hash GROUP BY 2-phase spill (2000 tuple calibration, 50% selectivity).
- Hash join partition count computed upfront — **no mid-build spill**.
- Parallel: single global named pool `"parallel-query"`, lock-free CAS reservation, log auto-degree.
- `arithmetic.c` owns 22 JSON scalar functions + `SLEEP()` (server-thread `usleep`)
- `DB_NUMERIC` is **16-byte two's-complement big-integer** (NOT BCD)
- DBLink password crypto = time-seeded XOR **obfuscation** (not a cipher)
- AND predicate short-circuits on V_FALSE, **NOT V_UNKNOWN** (correct 3VL)
- `qdata_evaluate_generic_function` is dead stub
- `scan-json-table` re-evaluates RapidJSON Pointer per row (no path cache)
- New SQL function registration = `qdata_evaluate_function` switch in `query_opfunc.c`

### Parallel query internals (post-round-5 deep dive)
- CAS reservation: `compare_exchange_weak` in-place update on failure; `push_task` fetch_add(release) pairs with `wait_workers` acquire
- MPMC slot ABA: sequence cycles `i → i+cap → i+2·cap`; dual CAS (enqueue expects `pos`, dequeue expects `pos+capacity`) + separate `ready` bool
- `atomic_instnum` uses `fetch_add` (over-emit tolerated)
- `err_messages::move_top_error_message_to_this()` SWAPS thread-local error into shared list
- `REGISTER_WORKERPOOL` at static-init; `call_once` failure is permanent
- Worker reservation via `try_reserve_workers(N)` returns 0 on contention (non-blocking)
- Parallel query-executor supports nested parallelism via parent-executor ctor (borrows pool)
- heap-scan trace uses Jansson JSON aggregator; query-executor trace uses XASL_STATS struct

### Storage / transactions
- Buffer pool: **3-zone LRU** (hot/buffer/victim); only zone 3 evictable.
- DWB recovery → WAL redo → ARIES (analysis/redo/undo).
- WAL ordering enforced inside `pgbuf_flush_with_wal` (not in callers).
- LOB external delete uses `LOG_POSTPONE` (not WAL).
- B-tree dispatch parameterized by **18 `btree_op_purpose` values**.
- `wait_for_graph.c` is dead code (gated `ENABLE_UNUSED_FUNCTION`); deadlock detection actually inside `lock_manager.c` — **contradicts** [[cubrid-AGENTS]] claim.
- Vacuum lives in `src/query/`, not `src/transaction/`.

### Conventions
- C error model in C++ code (no exceptions, no RAII for memory). [[Error Handling Convention]]
- `memory_wrapper.hpp` MUST be last include (architectural — avoids glibc placement-new conflict).
- Adding error code touches 6 files. CCI gets a 7th place (drift risk flagged in [[components/dbi-compat]]).
- `DB_TYPE` enum is ABI-frozen on disk + XASL stream — append after `DB_TYPE_JSON=40` only.
- `PT_NODE` function tables are ordinal-indexed (silent crash on misorder).

### Sessions / monitoring
- Sessions = zero-hash hot path via `thread_p->conn_entry->session_p` cached pointer.
- `@vars` and session params NOT rolled back on transaction abort.
- Each session owns its own page-buffer LRU zone.
- All cubmonitor stats are always-on (no sampling); sheets reused without zeroing → must compute snapshot delta.

### CDC / API
- CDC bypasses broker/CAS: `cubrid_log` opens raw CSS_CONN_ENTRY directly to cub_server, dedicated `NET_SERVER_CDC_*` sub-protocol. Only public CUBRID C API to do so. Single-threaded consumer (not thread-safe).
- `supplemental_log=1` + DBA membership required.

### Bundled
- `src/heaplayers/` = unmodified Emery Berger Heap Layers (Apache 2.0). `lea_heap.c` ≈ 181 KB Doug Lea dlmalloc. Engine surface = `HL_HEAPID` opaque handle only.

## Open follow-ups (from agent reports)
- **Flow pages worth filing**:
  - `pgbuf_fix → dwb_add_page → fileio_write` (page write lifecycle + WAL ordering)
  - B-tree insert with MVCC
  - `query-compile-flow` — one SELECT through all 6 parser passes
  - LOB write path: `lob_locator_add` → `es_create_file` → commit/rollback cleanup
  - End-to-end `NET_SERVER_QM_QUERY_EXECUTE` (client pack → CSS → server dispatch → executor → reply)
- **Source-code defects surfaced during round 5**:
  - `sort_copy_sort_param` declared in `px_sort.h` but implementation missing in `px_sort.c` — **resolved** by [[prs/PR-7011-parallel-index-build|PR #7011]] (OPEN); implementation lives at `external_sort.c:4344-4471` (next to consumer, not in `px_sort.c`)
  - `TASK_QUEUE_SIZE_PER_CORE = 2` constant defined but `thread_create_worker_pool` passes `1`
  - `reset_queue` epoch-bump invariant unclear — fires only when `pos % capacity == 0 && pos != 0`
- **Component pages**: `slotted_page`, `xasl_cache`, `query_opfunc`, `vacuum.c` deeper dive, `trigger_manager`, `work_space` (MOP cache), `transform.c`
- **Contradictions to file with `[!contradiction]` callouts**:
  - `wait_for_graph.c` ownership claim in [[cubrid-AGENTS]] vs reality (dead code)
  - `strict_warnings` listed for `src/debugging/` but file absent
- **Submodule ingests** (still pending): [[modules/cubrid-cci|cubrid-cci]], [[modules/cubrid-jdbc|cubrid-jdbc]], [[modules/cubridmanager|cubridmanager]]
- **Build / config / data dirs**: `cmake/`, `conf/`, `3rdparty/`, `locales/`, `timezones/`, `msg/`, `docs/`, `debian/`, `win/`, `contrib/`, `tests/`, `unit_tests/` (deeper than the existing top-level page)

## Active Threads
- Obsidian Git auto-commit + auto-push every 30 min → branch `cubrid1` on `xmilex-git/claude-obsidian`.
- Vault uses **Mode B** (codebase). 23 src/ subsystem sections in [[components/_index]], 21 CUBRID source summaries in [[sources/_index]].
- Original LLM-Wiki seed pages (concepts/entities about claude-obsidian itself) are retained as meta-docs; CUBRID is the dominant content now.
