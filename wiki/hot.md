---
created: 2026-04-23
type: meta
title: "Hot Cache"
updated: 2026-05-07
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
2026-05-07 (latest). **Pending wiki-updates buffer reconciliation — 8 entries, branch `parallel_scan_all` HEAD `58fab454f`.** Drained `~/dev/cubrid/.claude/wiki-updates/pending.md` of CBRD-26722 / PR #7062 divergences accumulated since the prior 2026-04-29 ingest. Branch HEAD advanced 6 commits (`7fdb82099` → `58fab454f` via `05d091c66` / `f74891494` / `fc1b51091` / `d117dd946`). PR #7062 remains OPEN — no PR-reconciliation, no baseline bump.

- [[prs/PR-7062-parallel-scan-all-types]] — three updates: (1) Behavioral § "Index scan — mutex-guarded leaf chain" rewritten as **per-range vertical descent + drain CV** (multi-range queries no longer use the leaf chain across range boundaries; transitions wait on `m_advance_cv` until `m_active_workers==0`; new HPP fields `m_active_workers`, `m_pending_advance_idx`, `m_advance_in_progress`, `m_advance_cv`); (2) range conversion bullet retitled to `input_handler_index::convert_all_key_ranges` — three-step pipeline (truncation collapse → DESC swap → storage-order sort); (3) trace-counter parity callout — `key_qualified_rows` and `read_rows` count visible OIDs per `m_slot_oids.size()` (parallel `:731-732`, serial `scan_manager.c:6279`) — pre-fix per-slot increments produced wildly wrong trace numbers on KEYLIST/RANGELIST + lookup queries while final row counts stayed correct. `head_sha` / `last_reingested_head` advanced to `58fab454f`.
- [[components/parallel-index-scan]] — four updates: (1) `public_api` block rewritten with new 4-arg / 3-arg signatures (`descend_to_first_leaf(thread_p, worker_scan_id, range_idx, out_leaf)`, `release_leaf_and_maybe_advance(thread_p, worker_scan_id, local_advance_target)`) + 5 new accessors; `get_next_leaf_with_fix` renamed to `get_next_page_with_fix`; (2) "Differences from parallel-list-scan" Partition-strategy row split single-range vs multi-range; (3) **new section "Vertical descent — serial parity (post `58fab454f`)"** — decision table closed-bound (`btree_search_nonleaf_page` direct call) vs open-bound (inlined `btree_find_boundary_leaf` walk); bound-case parity matrix INF_LT/GT_INF/GT_LT; `part_key_desc` swap parity (`btree.c:15972-15981` serial vs `convert_all_key_ranges:179-192` parallel); divergence table for cursor retention, strict-greater enforcement, range pre-sort, range crossing, cross-leaf state; (4) **new section "Error model — fail-loud, no `er_clear` (post `58fab454f`)"** — descent helpers preserve `er_set` payloads end-to-end, structurally fixing the silent-NULL `pgbuf_fix` class of bug from Invariant 3.
- [[components/btree]] — one update: **new section "Reused by parallel index scan"** documenting the asymmetric exposure (`btree_search_nonleaf_page` public in `btree.h:906` post commit `fc1b51091`; `btree_find_boundary_leaf` still file-static, mirrored inline at `px_scan_input_handler_index.cpp:340-378`). `[!gap]` callout suggests future export of `btree_find_boundary_leaf` would eliminate the second direct parser of the non-leaf record byte layout.

## Earlier Updates
2026-04-29. **CBRD-26722 knowledge dump ingest — parallel index scan on partitioned tables.** Branch-WIP capture from `parallel_scan_all` HEAD `7fdb82099` (4 commits beyond the prior PR #7062 review snapshot `0f8a107bb`). Two new pages:

- [[components/parallel-index-scan]] (address `c-000006`, `status: branch-wip`) — distilled four-invariant fix for the `ER_PT_EXECUTE(-495)`-with-empty-error crash that blocked parallel index scan on partitioned tables. **Invariant 1 (C1 `67e0eb852`)** — `PARALLEL_INDEX_SCAN_ID` reshaped as a layout superset of `INDX_SCAN_ID` with paired `offsetof` `static_assert`s in `scan_manager.h:170-313`; `PARALLEL_HEAP_SCAN_ID` already followed this pattern, the original 3-field pisid did not and corrupted isid's first 24 bytes on every union flip. **Invariant 2** — `manager::close()` does destructor + free in one (calling `~manager()` after `close()` is double-free; the asymmetry vs RAII C++ is a common review trap). **Invariant 3 (C3 `9185c1aae`)** — XASL stream is compile-time frozen, so `qexec_init_next_partition:9073`'s live `spec->indexptr->btid` update is invisible to `clone_xasl`'d workers; without override, `pgbuf_fix(Root_vpid)` returns NULL on the parent class with no `er_set`, surfacing as the wrapped `ER_PT_EXECUTE(-495)` at `qexec_execute_mainblock:16581`. Worker `task::initialize` now overrides via `m_input_handler->get_indx_info()`. **Invariant 4** — `m_btid_int.sys_btid` is NULL until first worker's `descend_to_first_leaf` (lazy via PR #7062's latch-couple `0f8a107bb`); use `m_btid` / `get_indx_info()` for safe BTID access at promote time. **Invariant 5 (C4 `7fdb82099` HEAD)** — final-iteration parent-class re-open at `qexec_init_next_partition` rolls `scan_id->type` back to `S_INDX_SCAN`; pisid `trace_storage` survives the flip via C1's superset layout, but `query_dump.c:3093, 3553` had to grow the OR pattern (`S_PARALLEL_INDEX_SCAN || S_INDX_SCAN`) that already existed on the HEAP side at `:3540`. Cross-cutting: per-partition `trace_storage` orphan at `scan_try_promote` line 1560 — known cosmetic limitation. Diagnosis playbook for `ER_PT_EXECUTE(-495)` recorded.
- [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables]] (address `c-000007`) — source-trail page with provenance, file map, and acceptance-criteria evidence.

[[prs/PR-7062-parallel-scan-all-types]] re-anchored: `head_sha` → `7fdb82099`, stats line "87 commits" → "91 commits", "Branch-WIP companion pages" extended (now 3 — added the parallel-index-scan cross-link), new "Commits after `0f8a107bb`" subsection commit-by-commit summarising C1–C4. PR remains OPEN — no PR-reconciliation, no baseline bump. No incidental baseline edits this round (every fact intertwined with branch-only `px_scan/` code or latch-couple).

**Earlier 2026-04-29. `log_sysop_*()` family ingest.** New component page [[components/log-sysop]] (address `c-000005`) covering the system-operation logging primitive in `src/transaction/log_manager.c:3563-4178` + `log_sysop_do_postpone` at `:8189`. 18-function family (9 public, 9 static), conceptual nested-TX-within-TX model with `LOG_TDES.topops` stack, six `LOG_SYSOP_END_TYPE` subtypes, `LOG_REC_SYSOP_END` payload union (`log_record.hpp:304-324`), atomic-sysop recovery semantics (`LOG_SYSOP_ATOMIC_START` rectype 50 → eager rollback before redo phase), postpone interaction with `LOG_SYSOP_START_POSTPONE` rectype 18 cache, vacuum-tdes redirection rule centralised in `log_sysop_get_tran_index_and_tdes`, `lock_topop` mutex, checkpoint-trigger at outermost-end-final, empty-sysop short-circuit + asymmetric ban on empty logical-end variants, failure-mode table. Cross-linked from [[components/log-manager]] and [[components/recovery]]; registered in [[components/_index]]. Also: added `PreToolUse:Edit` hook to `hooks/hooks.json` that auto-reads target file before edits.

**Earlier 2026-04-29. PR #7062 re-ingest — re-anchored to HEAD `0f8a107bb`.** External review doc at `/home/cubrid/dev/cubrid/.claude/knowledge/pr_7062_code_review.md` was rewritten by the user to reflect 22 commits beyond the original `c28c5945a` snapshot, and re-ingested via the wiki-ingest skill. [[prs/PR-7062-parallel-scan-all-types]] page updated in place: frontmatter `head_sha` → `0f8a107bb…`, `base_sha` corrected to baseline `0be6cdf6e…`, stats refreshed (87 commits / +6,061 / −1,753), aggregate-count fixed (13, not 11), new "Commits after `c28c5945a`" subsection grouping the 22 follow-on commits by theme (greptile P1 fixes, INDEX/LIST single-thread fallback, two-parameter split + 137-line `qo_apply_parallel_index_scan_threshold` cost-gate, BUILDVALUE_OPT merge from `parallel_buildvalue_heap`, double-fix elimination in both list and index paths, latch-couple of root→leaf descent + leaf chain, lazy CAS membuf claim decoupled from idx 0). PR remains OPEN — no PR-reconciliation applied, no baseline bump. Reconciliation Plan shape unchanged (post-`c28c5945a` deltas already covered via existing plan items + branch-WIP companion pages). No new component pages or incidental enhancements (the prior 2026-04-29 round drained the baseline-truth surface — all 4 incidental targets remain current).

**Earlier 2026-04-29. Pending wiki-updates ingest from CUBRID workspace.** Processed 11 divergence entries from `/home/cubrid/dev/cubrid/.claude/wiki-updates/pending.md` (logged on branch `parallel_scan_all`, head `0f8a107bb`, [CBRD-26722]). Two new branch-WIP pages: [[components/parallel-list-scan]] (input handler + slot iterator: static slice partitioning, lazy CAS-claimed membuf, per-page tfile, **TWO** silent-skip sentinels — overflow filter at page level + `tuple_count == 0` race at slot level) and [[flows/parallel-list-scan-open]] (end-to-end open sequence; the `run_jobs()` join barrier is the only thing closing the silent-skip race). Three baseline-truth incidental enhancements: [[components/list-file]] (membuf-only-on-base invariant in `qfile_collect_list_sector_info`; `qfile_close_list` contract — terminator + unfix only, no flush; QFILE page-header layout — no writer marker, only `tuple_count` sentinels), [[components/file-manager]] (`file_get_all_data_sectors` temp-file-vs-permanent-file gate at `file_manager.c:12608-12622` — temps keep all sectors in partial-FTAB, load-bearing for parallel temp scans), [[components/parallel-query-task]] (list_id save/restore stack-local dance in parallel-aptr teardown). [[prs/PR-7062-parallel-scan-all-types|PR #7062]] page extended with branch-WIP companion-pages cross-link + post-write supersession callout (`0f8a107bb` decoupled membuf from idx 0 — earlier "Worker 0 always becomes the membuf-worker" claim is now stale). No baseline bump (PR #7062 OPEN).

**Earlier 2026-04-28. PR #6981 ingest + baseline bump.** [[prs/PR-6981-parallel-hash-join-sector-split]] (MERGED, case c, 8 files +384/−106, merge `0be6cdf6`). Replaces `HASHJOIN_SHARED_SPLIT_INFO::scan_mutex` + `(scan_position, next_vpid)` cursor in parallel hash join's *split* phase with lock-free sector-bitmap distribution: `std::atomic<int> next_sector_index` + per-worker `__builtin_ctzll` bitmap walk + single CAS-claim for membuf (`std::atomic<bool> membuf_claimed`). New generic helper `qfile_collect_list_sector_info` (in `list_file.c`) harvests sectors from a `QFILE_LIST_ID` *and* its `dependent_list_id` chain into a flat `QFILE_LIST_SECTOR_INFO` (sectors + parallel `tfiles[]` array — required because dependent-list pages must be released against their own `QMGR_TEMP_FILE *`). Per-worker `m_current_tfile` recorded by `get_next_page` and used for all page-release + overflow-chain `qmgr_get_old_page` calls in `execute()`. Overflow continuation pages skipped on bitmap walk (they share the start page's sector via `qfile_allocate_new_ovf_page`). Drop-in correctness fix in both serial fallback (`hjoin_split_qlist`) and parallel `split_task::execute`: `qfile_destroy_list + free → qfile_truncate_list + retain LIST_ID` so a mid-flush `qfile_append_list` failure can't double-free. Baseline bumped `cc563c7f` → `0be6cdf6`. 4 component pages reconciled (parallel-hash-join, parallel-hash-join-task-manager [major], list-file, file-manager). 1 incidental enhancement (continuation-page bitmap-double-consumption mechanism in [[components/list-file]]).

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
