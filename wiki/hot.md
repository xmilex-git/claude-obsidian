---
created: 2026-04-23
type: meta
title: "Hot Cache"
updated: 2026-04-28
tags:
  - meta
  - hot
status: active
---

# Recent Context

## CUBRID Baseline Commit
**`0be6cdf6ee66f9fa40a84874004d9b4e3a642ff0`** — all wiki claims, file paths, line numbers reflect this commit of `~/dev/cubrid/`. Before any new ingest, check repo HEAD; if newer, compute `git log <baseline>..HEAD -- <path>` and reconcile affected wiki pages before writing. Protocol: `CLAUDE.md` § "CUBRID Baseline Commit".

> [!update] Bumped 2026-04-28 from `cc563c7f` → `0be6cdf6` via [[prs/PR-6981-parallel-hash-join-sector-split|PR #6981]] reconciliation. Direct child of prior baseline on `develop` (case c, single squash-merge).
> Earlier 2026-04-27 (later): `65d69154` → `cc563c7f` via [[prs/PR-7011-parallel-index-build|PR #7011]] reconciliation (case c).
> Earlier 2026-04-27: `175442fc` → `65d69154` via [[prs/PR-7049-parallel-buildvalue-heap|PR #7049]] (case c).

## PR Ingest
**On-demand only, user-specified only.** Do not autonomously scan, poll, or batch. All PR states are ingestable — behavior differs: merged-case-c → reconcile + bump now; merged-case-a/b → retroactive doc only; open/draft → write Reconciliation Plan (do NOT edit component pages for PR-induced changes); closed-unmerged → doc as abandoned. **Deep code analysis required** every time (read baseline source files, not just the diff). **Incidental Knowledge Enhancement expected** every time: facts about baseline code that are missing/wrong/incomplete in the wiki get added to component/source pages immediately, regardless of PR state. Deferred plan execution via "apply reconciliation for PR #NNNN". Template: `_templates/pr.md`; index: [[prs/_index|PRs]]; protocol: `CLAUDE.md` § "PR Ingest (user-specified only, all states accepted, code analysis required)".

## Last Updated
2026-04-28. **PR #6981 ingest + baseline bump.** [[prs/PR-6981-parallel-hash-join-sector-split]] (MERGED, case c, 8 files +384/−106, merge `0be6cdf6`). Replaces `HASHJOIN_SHARED_SPLIT_INFO::scan_mutex` + `(scan_position, next_vpid)` cursor in parallel hash join's *split* phase with lock-free sector-bitmap distribution: `std::atomic<int> next_sector_index` + per-worker `__builtin_ctzll` bitmap walk + single CAS-claim for membuf (`std::atomic<bool> membuf_claimed`). New generic helper `qfile_collect_list_sector_info` (in `list_file.c`) harvests sectors from a `QFILE_LIST_ID` *and* its `dependent_list_id` chain into a flat `QFILE_LIST_SECTOR_INFO` (sectors + parallel `tfiles[]` array — required because dependent-list pages must be released against their own `QMGR_TEMP_FILE *`). Per-worker `m_current_tfile` recorded by `get_next_page` and used for all page-release + overflow-chain `qmgr_get_old_page` calls in `execute()`. Overflow continuation pages skipped on bitmap walk (they share the start page's sector via `qfile_allocate_new_ovf_page`). Drop-in correctness fix in both serial fallback (`hjoin_split_qlist`) and parallel `split_task::execute`: `qfile_destroy_list + free → qfile_truncate_list + retain LIST_ID` so a mid-flush `qfile_append_list` failure can't double-free. Baseline bumped `cc563c7f` → `0be6cdf6`. 4 component pages reconciled (parallel-hash-join, parallel-hash-join-task-manager [major], list-file, file-manager). 1 incidental enhancement (continuation-page bitmap-double-consumption mechanism in [[components/list-file]]).

**Earlier 2026-04-27 (later) — PR #7011 reconcile + baseline bump.** PR merged at 05:20Z as `cc563c7`; reconciliation plan promoted to "Pages Reconciled" and applied to all 7 component pages with `[!update]` callouts citing the merge sha. Baseline bumped `65d69154` → `cc563c7f`.

**Earlier 2026-04-27 — PR #7049 ingest + baseline bump.** [[prs/PR-7049-parallel-buildvalue-heap]] (MERGED, case c, 9 files +424/−158, merge `65d6915`). Extends parallel heap scan BUILDVALUE_PROC fast path from COUNT-only to 12 aggregates (COUNT/MIN/MAX/SUM/AVG/STDDEV*/VAR*); enum rename `RESULT_TYPE::COUNT_DISTINCT` → `BUILDVALUE_OPT`. Key engineering: **two-heap dance** (workers in heap 0, main re-clones into private heap before downstream `pr_clear_value`); `qdata_aggregate_accumulator_to_accumulator` reused for per-worker partial merge; STDDEV/VAR use sum-of-x + sum-of-x² two-slot accumulator; MIN/MAX(DISTINCT) shortcut. Baseline bumped `175442fc` → `65d69154`. 7 component pages reconciled.

**Earlier 2026-04-27 — Manual ingest session.** Cataloged the full CUBRID 11.4 English User Manual (`/home/cubrid/cubrid-manual/en/` — 119 RST files, ~88K lines, 37 MB) into 22 `cubrid-manual-*` source pages. Strategy: catalog + enhance (NOT per-file ingest — 1-line summary per RST file, key facts inline, cross-refs back to RST tree). Hub: [[sources/cubrid-manual-en-overview]]. **4 parallel-agent dispatch** for sql/, admin/, api/, pl/ clusters. **Top-7 incidental enhancements applied** to component pages.

**Prior session (2026-04-26):** PR #7011 deep ingest (parallel index build, OPEN, 9 files). [[prs/PR-7011-parallel-index-build]] with full Reconciliation Plan. Resolved baseline gap re: `sort_copy_sort_param` location.

**Common themes across recent PRs (#6981, #7049, #7011, #6911):** parallel-query subsystem expansion, reusing existing primitives. PR #6981 + #6911 both port `file_get_all_data_sectors` sector pre-split (originally heap-only) to other parallel subsystems. PR #7049 + #7011 generalize aggregate / scan dispatch beyond their original narrow scope. **Cross-PR primitive watch: `file_get_all_data_sectors` is now consumed by 4 paths** — parallel heap scan (PR #6911), parallel index build via `SORT_INDEX_LEAF` (PR #7011), and parallel hash join split phase via `qfile_collect_list_sector_info` (PR #6981).

**Prior session (2026-04-24):** Lint + legacy cleanup. Filed `lint-report-2026-04-24`. 18 pre-CUBRID pages moved into `wiki/_legacy/`.

**Prior session (2026-04-23):** CUBRID deep-dive rounds 1–5 finished. 150 component pages, 34 source summaries, 246 total wiki md.

## Wiki shape
- `wiki/components/` (157 pages) — one section per CUBRID subsystem
- `wiki/sources/` (53 pages + _index) — 32 source-tree + 21 manual-catalog
- `wiki/modules/`, `wiki/decisions/`, `wiki/dependencies/`, `wiki/flows/`, `wiki/prs/` — Mode B scaffold
- Hub pages: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Dependency Graph]], [[Key Decisions]]
- Concept pages: [[Query Processing Pipeline]], [[Build Modes (SERVER SA CS)]], [[Memory Management Conventions]], [[Error Handling Convention]], [[Code Style Conventions]]
- **CUBRID 11.4 User Manual catalog**: [[sources/cubrid-manual-en-overview]] hub + 21 section pages (see [[sources/_index]])

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

### Parallel query internals (post-round-5 deep dive + PR #6981/#7049/#7011)
- CAS reservation: `compare_exchange_weak` in-place update on failure; `push_task` fetch_add(release) pairs with `wait_workers` acquire
- MPMC slot ABA: sequence cycles `i → i+cap → i+2·cap`; dual CAS (enqueue expects `pos`, dequeue expects `pos+capacity`) + separate `ready` bool
- `atomic_instnum` uses `fetch_add` (over-emit tolerated)
- `err_messages::move_top_error_message_to_this()` SWAPS thread-local error into shared list
- `REGISTER_WORKERPOOL` at static-init; `call_once` failure is permanent
- Worker reservation via `try_reserve_workers(N)` returns 0 on contention (non-blocking)
- Parallel query-executor supports nested parallelism via parent-executor ctor (borrows pool)
- heap-scan trace uses Jansson JSON aggregator; query-executor trace uses XASL_STATS struct
- **Sector pre-split primitive (`file_get_all_data_sectors`)** is the standard distribution mechanism across parallel paths: parallel heap scan (PR #6911), parallel index build `SORT_INDEX_LEAF` (PR #7011), and parallel hash join split phase via `qfile_collect_list_sector_info` (PR #6981 — adds dependent-list-chain merge + parallel `tfiles[]` array). Workers claim sectors via `next_sector_index.fetch_add(1, relaxed)`, walk per-sector 64-page bitmaps with `__builtin_ctzll`. Membuf pages (in-memory portion) handled by single CAS-claim. Overflow continuation pages skipped via `QFILE_GET_TUPLE_COUNT == QFILE_OVERFLOW_TUPLE_COUNT_FLAG` (continuation pages share start page's sector bitmap).

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
- **Parallel hash join (post PR #6981)**: split phase is now lock-free for page distribution. Per-thread `m_current_tfile` recording is the new pattern to remember when working with `dependent_list_id` chains — base-list and dependent-list pages live in the same `sector_info->sectors` array but must be released against their own `QMGR_TEMP_FILE *` (carried in the parallel `sector_info->tfiles[]`). The `qfile_destroy_list → qfile_truncate_list` correctness fix landed in both serial (`hjoin_split_qlist`) and parallel paths simultaneously — keep them symmetric in future edits.
- **Parallel heap scan (post PR #7049)**: BUILDVALUE_OPT fast path now covers 12 aggregates (was 2). Pattern to remember: per-worker partial accumulator in heap 0 → main-thread `qdata_aggregate_accumulator_to_accumulator` merge in heap 0 → main-thread `read` re-clones into private heap. STDDEV/VAR uses `value` (sum x) + `value2` (sum x²) two-slot accumulator. MIN/MAX(DISTINCT) bypasses the per-thread DISTINCT list entirely. Eligibility checked by `is_buildvalue_opt_supported_function` whitelist in `px_heap_scan_checker.cpp`.

- **Flow pages worth filing**:
  - `pgbuf_fix → dwb_add_page → fileio_write` (page write lifecycle + WAL ordering)
  - B-tree insert with MVCC
  - `query-compile-flow` — one SELECT through all 6 parser passes
  - LOB write path: `lob_locator_add` → `es_create_file` → commit/rollback cleanup
  - End-to-end `NET_SERVER_QM_QUERY_EXECUTE` (client pack → CSS → server dispatch → executor → reply)
- **Source-code defects surfaced during round 5**:
  - `sort_copy_sort_param` declared in `px_sort.h` but implementation missing in `px_sort.c` — **resolved** by [[prs/PR-7011-parallel-index-build|PR #7011]] (merged); implementation lives at `external_sort.c:4344-4471` (next to consumer, not in `px_sort.c`)
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
