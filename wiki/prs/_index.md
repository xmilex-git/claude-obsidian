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
- [[prs/PR-6911-parallel-heap-scan-io-bottleneck|PR #6911 — Reduce I/O bottleneck when parallel heap scan]] (CBRD-26615, merged 2026-03-27, case-b retroactive) — replaces per-page mutex handoff with upfront sector allocation; pgbuf API left unchanged after review.

---

## Relationship to other pages

- **[[decisions/_index|Decisions]]** — when a PR represents a major design choice, a companion ADR is filed under `decisions/` citing the PR page.
- **[[components/*]]** — PR ingest updates component pages with `> [!update]` callouts that cite the PR number and merge commit.
- **[[sources/*]]** — file-level source pages get the same `> [!update]` treatment.
- **[[log]]** — every PR ingest that bumps the baseline produces a `baseline-bump` log entry.
