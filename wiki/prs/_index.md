---
type: index
title: "CUBRID PRs"
created: 2026-04-24
updated: 2026-04-24
tags:
  - index
  - cubrid
  - pr
status: active
related:
  - "[[index]]"
  - "[[log]]"
  - "[[Key Decisions]]"
  - "[[decisions/_index]]"
---

# CUBRID Pull Requests

Navigation: [[index]] | [[modules/_index|Modules]] | [[components/_index|Components]] | [[decisions/_index|Decisions]]

Documented CUBRID upstream PRs (`CUBRID/cubrid`) — **all states accepted**, one page per PR. Each page captures:
- Motivation + summary of the proposed/landed change
- Files + structural + behavioral impact from **deep code analysis** (not just diff reading — the source files themselves)
- Authoritative review discussion (design rationale only)
- **Reconciliation Plan** for open/draft PRs (what wiki pages would change on merge, with concrete before/after excerpts — executable later without re-reading the PR)
- **Pages Reconciled** for merged PRs newer than baseline (what was edited, which callouts added)
- **Incidental wiki enhancements** — baseline-truth facts surfaced during analysis that were missing from the wiki and have been added directly to component/source pages (orthogonal to PR-reconciliation; applies to every PR state)
- Baseline-bump before/after hashes

**User-specified only.** Claude does not scan, poll, or batch PRs on its own initiative. See `CLAUDE.md` § "PR Ingest (user-specified only, all states accepted, code analysis required)" for the full protocol and the state-to-behavior matrix.

Filename convention: `PR-NNNN-short-slug.md` where `NNNN` is the upstream PR number.

### State handling at a glance

| PR state | PR page | PR-reconciliation | Reconciliation Plan | Incidental enhancements | Baseline bump |
|---|---|---|---|---|---|
| merged, baseline ancestor (case a/b) | yes | no | n/a | yes | no |
| merged, newer than baseline (case c) | yes | **yes, immediately** | (promoted to Pages Reconciled) | yes | **yes** |
| merged, divergent (case d) | stop, ask user | — | — | — | — |
| open / approved | yes | no | **yes, written** | yes | no |
| draft | yes | no | yes | yes | no |
| closed-unmerged | yes | no | no | yes (if any) | no |

Deferred plan execution: user later says "apply reconciliation for PR #NNNN" → plan is read, revalidated against current state, executed, and `reconciliation_applied` flag set.

## Ingested PRs

<!-- All states accepted (merged, open, draft, closed-unmerged). Newest first. -->

- [[prs/PR-6753-optimizer-histogram-support|PR #6753 — Add Optimizer Histogram Support]] (CBRD-26202, **OPEN**, Reconciliation Plan written) — new `src/optimizer/histogram/` subsystem (6 files, 3300 LOC), `_db_histogram` catalog + `db_histogram` view, `ANALYZE TABLE … UPDATE|DROP HISTOGRAM` and `SHOW HISTOGRAM` DDL, MCV+equi-depth buckets with `HST1` blob format, Poisson sampling-scan weight, new `default_histogram_bucket_count` sysprm (default 300). 5 incidental wiki enhancements. 47 supplementary findings beyond existing bot reviews — incl. HISTOGRAM/BUCKETS reserved-word breaking change, `oid_Histogram_class` never populated (disabled MVCC-skip), privilege gap, TRUNCATE leak, unload regression.
- [[prs/PR-7062-parallel-scan-all-types|PR #7062 — Expand parallel heap scan to parallel scan (index, heap, list)]] (CBRD-26722, **OPEN**, Reconciliation Plan written) — generalises `parallel_heap_scan` namespace to `parallel_scan::manager<RT, ST>` over heap/list/index; new input handlers + slot iterators for list & index; `XASL_SNAPSHOT × LIST/INDEX` blocked by checker; new `parallel_scan_page_threshold` system param; hint surface unchanged via `NO_PARALLEL_HEAP_SCAN → NO_PARALLEL_SCAN` rename. 1 incidental wiki enhancement on [[components/xasl]].
- [[prs/PR-7011-parallel-index-build|PR #7011 — Support parallel index build]] (CBRD-26678, **OPEN**, 5 approvals, Reconciliation Plan written) — parallelizes CREATE INDEX heap-scan + sort phase by reusing `SORT_EXECUTE_PARALLEL` infrastructure with new `SORT_INDEX_LEAF` discriminator. Generalizes `parallel_heap_scan::ftab_set` → `parallel_query::ftab_set`. Promotes `SORT_ARGS` from file-static to public header. Adds `btree_sort_get_next_parallel` + `get_next_vpid` per-thread sector iterator with `pgbuf_ordered_fix`+`pgbuf_replace_watcher` page-fix protocol. Per-worker XASL filter/func-index re-deserialization. Sysop ownership delegation: SERVER_MODE → sort layer, SA_MODE → btree_load.c. log₄ tree-merge fan-in with empty-worker skip. `btree_create_file` hoisted to merge phase. **Resolves the previously-flagged `sort_copy_sort_param` baseline gap.** 1 incidental wiki enhancement on hot cache.
- [[prs/PR-6443-system-catalog-information-schema|PR #6443 — Improve and refactor System Catalog for Information Schema]] (CBRD-25862, MERGED 2025-12-12, case-b absorbed) — rollup over `feature/system-catalog` (14+ sub-commits): adds time + policy columns to 8 system catalog classes (`_db_class`, `_db_index`, `_db_partition`, `_db_serial`, `_db_stored_procedure`, `_db_synonym`, `_db_server`, `_db_trigger`); split `is_system_class` flag into a separate `flags` catalog column at the row layer (heap layout unchanged); naming convention `_db_*` table / `db_*` view (auth classes exempted); `db_authorizations` removed entirely; new indexes on `_db_class.class_of`, `_db_data_type.(type_id,type_name)`, `_db_collation.coll_id`; PL/CSQL dependency tracking added through `compile_response::dependencies`. 1 incidental wiki enhancement on system-catalog. **38 supplementary findings**, including: NO migration path (boot SIGABRT on old DBs), login fails on un-migrated DBs (`is_loginable` lookup), inverted if-error logic in trigger register/unregister, `unloaddb` missing `start_val` (round-trip-lossy), `sql_data_access` reserved-but-broken (no parser path), wire-format change without protocol-version bump, `disable_login` dead code, `change_serial_owner` removal incomplete, defense-in-depth lost via AU_DISABLE proliferation.
- [[prs/PR-6689-bcb-mutex-to-atomic-latch|PR #6689 — Replace BCB mutex lock with atomic latch]] (CBRD-26425, MERGED into `feature/refactor-pgbuf`, absorbed in baseline via sibling PR #6704 — case b retroactive doc) — replaces per-BCB `pthread_mutex_t` with packed 64-bit `std::atomic<uint64_t>` holding `{latch_mode, waiter_exists, fcnt}`; adds lockfree RO fast path (`pgbuf_lockfree_fix_ro` + `pgbuf_search_hash_chain_no_bcb_lock` + `pgbuf_lockfree_unfix_ro`); hoists `m_is_private_lru_enabled` + `m_holder_anchor` to `cubthread::entry`. 2 incidental wiki enhancements. 17 supplementary findings including 5 latent correctness bugs (double-lock on CAS retry, fcnt<0 never CAS-clamped, lockfree path bypasses hot-page registration).
- [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911 — Reduce I/O bottleneck when parallel heap scan]] (CBRD-26615, merged 2026-03-27, case-b retroactive) — replaces per-page mutex handoff with upfront sector allocation; pgbuf API left unchanged after review.

---

## Relationship to other pages

- **[[decisions/_index|Decisions]]** — when a PR represents a major design choice, a companion ADR is filed under `decisions/` citing the PR page.
- **[[components/*]]** — PR ingest updates component pages with `> [!update]` callouts that cite the PR number and merge commit.
- **[[sources/*]]** — file-level source pages get the same `> [!update]` treatment.
- **[[log]]** — every PR ingest that bumps the baseline produces a `baseline-bump` log entry.
