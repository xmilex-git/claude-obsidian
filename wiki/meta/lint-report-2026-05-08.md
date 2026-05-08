---
type: meta
title: "Lint Report 2026-05-08"
created: 2026-05-08
updated: 2026-05-08
tags:
  - meta
  - lint
status: developing
---

# Lint Report: 2026-05-08

## Summary

- Pages scanned: 318 total (280 non-legacy)
- Issues found: 29 (4 critical, 13 warnings, 12 suggestions)
- Auto-fixed: 0 (report only)
- Scope: full vault, focus on 2026-05-08 PR ingest pass

---

## Critical (must fix)

### C1 — `updated:` frontmatter not bumped on 6 recently-edited pages

All six component pages reconciled during the 2026-05-08 PR ingest pass still carry `updated: 2026-04-23` in their frontmatter, despite receiving new sections today.

Affected pages (all have `updated: 2026-04-23`):
- `wiki/components/lock-manager.md` — received "Initialization (since PR #6930)" section and finalize-order gap fill
- `wiki/components/deadlock-detection.md` — received "TWFG and Victim Storage (since PR #6930)" update callout
- `wiki/components/db-value.md` — received `db_get_char` accessor entry and key-insight callout
- `wiki/components/base.md` — received "UTF-8 counting and validation" subsection
- `wiki/components/cas.md` — received CAS trailing-NUL key-insight callout
- `wiki/components/loaddb-executor.md` — received domain-precision key-insight callout

Fix: set `updated: 2026-05-08` on all six pages.

---

### C2 — `wiki/index.md` carries stale baseline commit and stale `Last updated` date

Line 32 of `wiki/index.md`:
```
Last updated: 2026-04-27 | Mode: B (CUBRID codebase) | Baseline: `65d69154`
```
The current baseline is `05a7befd`. The date is also four ingests stale. This is the vault's primary navigation page and its header is wrong.

Fix: update to `Last updated: 2026-05-08 | Baseline: \`05a7befd\``.

---

### C3 — DragonScale: two post-rollout pages missing `address:` field

Counter state: `8` (next address would be `c-000008`). Highest observed: `c-000007`.
Post-rollout pages checked: 7 (5 passing, 2 errors).

Both new PR pages created 2026-05-08 lack an `address:` field. Per the DragonScale rollout baseline (earliest addressed page `created: 2026-04-29`), all non-meta pages created on or after 2026-04-29 require an address.

- `wiki/prs/PR-6930-lock-manager-init-refactor.md` (created: 2026-05-08) — missing `address:`
- `wiki/prs/PR-7102-db-get-char-intl-cleanup.md` (created: 2026-05-08) — missing `address:`

Fix: run `./scripts/allocate-address.sh` twice and add the returned addresses to each page's frontmatter and to `.raw/.manifest.json` address_map. Do not auto-assign during lint.

---

### C4 — DragonScale: Semantic Tiling script fails on Python 3.6 (exit 1, TypeError)

`scripts/tiling-check.py --peek` exits 1 with:
```
TypeError: 'type' object is not subscriptable
  File "scripts/tiling-check.py", line 132: def parse_frontmatter(text: str) -> tuple[dict, str]:
```
The script uses PEP 585 built-in generics (`tuple[dict, str]`) which require Python 3.9+. The vault's system Python is 3.6.8. Per the skill spec, exit code 1 is "unexpected" and must be surfaced distinctly — this is not exit 10 (ollama unreachable) or exit 11 (model missing); it is a script runtime error that prevents tiling from running at all.

Semantic Tiling status: **disabled** (script crashes before any tiling check runs).

Fix (choose one): (a) add `from __future__ import annotations` at the top of `scripts/tiling-check.py` to defer annotation evaluation, or (b) install Python 3.9+ as `python3` on this host, or (c) replace `tuple[dict, str]` with `typing.Tuple[dict, str]` throughout the script for 3.6 compatibility.

---

## Warnings (should fix)

### W1 — `updated:` missing on 7 component pages

These component pages have no `updated:` field at all (distinct from C1's wrong-date issue):

- `wiki/components/execute-schema.md`
- `wiki/components/execute-statement.md`
- `wiki/components/query-cl.md`
- `wiki/components/query-dump.md`
- `wiki/components/query-fetch.md`
- `wiki/components/query-manager.md`
- `wiki/components/query-reevaluation.md`

Fix: add `updated: 2026-04-23` (their apparent creation date) to each.

---

### W2 — `updated:` missing on all 22 `cubrid-manual-*` source pages and 14 other source pages

All 22 manual catalog pages created 2026-04-27 lack `updated:`. Additionally 14 other source pages (`cubrid-3rdparty`, `cubrid-contrib`, `cubrid-msg`, `cubrid-query-scan-family`, `cubrid-src-*` family) lack `updated:`.

This is a bulk gap from the original ingest session that never set the field. Fix: set `updated: 2026-04-27` on the 22 manual pages; set `updated: 2026-04-23` on the remaining source pages.

---

### W3 — `status:` field missing on one source page

- `wiki/sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables.md` — frontmatter has no `status:` field.

Fix: add `status: branch-wip` (consistent with the page content, which explicitly notes it is a branch-WIP capture).

---

### W4 — `created:` field missing on 3 meta pages

- `wiki/meta/2026-04-15-release-report-session.md`
- `wiki/meta/2026-04-15-slides-and-release-session.md`
- `wiki/meta/boundary-frontier-2026-04-24.md`

Fix: add `created:` matching the date in the filename for each.

---

### W5 — `wiki/index.md` Concepts section uses bare names instead of `concepts/` paths

`wiki/index.md` links concept pages as `[[Query Processing Pipeline]]`, `[[Build Modes (SERVER SA CS)]]`, etc. These files actually live under `wiki/concepts/`. The Obsidian resolver finds them by filename uniqueness, but the lint script (and any strict resolver) reports them as dead because they don't resolve via path. The same applies to `[[CUBRID]]` (lives under `wiki/entities/`).

Affected index entries:
- `[[Query Processing Pipeline]]` → file at `wiki/concepts/Query Processing Pipeline.md`
- `[[Build Modes (SERVER SA CS)]]` → file at `wiki/concepts/Build Modes (SERVER SA CS).md`
- `[[Memory Management Conventions]]` → file at `wiki/concepts/Memory Management Conventions.md`
- `[[Error Handling Convention]]` → file at `wiki/concepts/Error Handling Convention.md`
- `[[Code Style Conventions]]` → file at `wiki/concepts/Code Style Conventions.md`
- `[[CUBRID]]` → file at `wiki/entities/CUBRID.md`

Fix: change to path-qualified links (`[[concepts/Query Processing Pipeline]]`, etc.) or accept that Obsidian's global resolver handles these correctly and note the discrepancy only for tooling.

Note: the same bare-name pattern appears in `wiki/components/base.md` (links `[[Error Handling Convention]]`, `[[Memory Management Conventions]]`, `[[Build Modes (SERVER SA CS)]]`) and throughout the component pages generally. This is a systemic pattern, not unique to index.md.

---

### W6 — `wiki/index.md` links `[[dashboard]]` and `[[Wiki Map.canvas]]` via stale paths

- `[[dashboard]]` resolves to `wiki/meta/dashboard.md` (file exists at that path). The bare name resolves correctly via Obsidian but the path-qualified check fails because there is no `wiki/dashboard.md`. Strictly, the frontmatter `related:` and navigation bar should use `[[meta/dashboard]]` for clarity.
- `[[Wiki Map.canvas]]` is a canvas file at `wiki/Wiki Map.canvas` (no `.md`). Canvas files are not `.md` and cannot be validated as wiki pages. This is cosmetic for Obsidian but breaks any lint tool.

Fix: update `[[dashboard]]` to `[[meta/dashboard]]` in index.md frontmatter and navigation; accept `[[Wiki Map.canvas]]` as an Obsidian-specific canvas link that lint tools will always flag.

---

### W7 — `wiki/meta/tiling-report-2026-04-24.md` has no frontmatter

`wiki/meta/tiling-report-2026-04-24.md` is the only file in `wiki/meta/` with no frontmatter at all.

Fix: add minimal frontmatter (`type: meta`, `title`, `created: 2026-04-24`, `updated: 2026-04-24`, `tags: [meta, tiling]`, `status: developing`).

---

### W8 — Address gap at `c-000002` (informational)

The address sequence runs `c-000001`, then `c-000003` through `c-000007`. The `c-000002` slot was consumed by the allocator during a test run (logged in `wiki/log.md` as "unassigned reservation; gap acceptable per spec") and has no assigned page. The manifest does not map `c-000002`. Per the skill spec this is an accepted gap, but it is documented here for completeness. No fix required unless the team wants to audit that the gap is intentional.

---

## Suggestions (worth considering)

### S1 — PR pages not listed in `wiki/index.md` PRs sub-index

`wiki/index.md` links `[[prs/_index|PRs]]` but does not enumerate individual PRs. The two new pages (`PR-6930`, `PR-7102`) are correctly registered in `wiki/prs/_index.md` and `wiki/log.md`. No fix required for correctness, but the index could optionally list the newest 3–5 PRs for discoverability.

---

### S2 — `updated:` on the PR ingest pages is stale relative to their actual modification date

`wiki/prs/_index.md` has `updated: 2026-05-08` (correct). Both new PR pages have `updated: 2026-05-08` (correct). No issue here — noted as confirming good practice.

---

### S3 — `lock-manager.md` section "Initialization (since PR #6930)" contains a file-static function documented as "reachable to in-tree callers"

The PR page (line ~205) states `lock_initialize_with_config` is `file-static` but "reachable to in-tree callers". This is a mild contradiction: a `static` C function is translation-unit private and not callable from other `.c` files. The intended meaning is that future refactors could expose it. Consider adding a clarifying note on `wiki/components/lock-manager.md` that this function is TU-private and any exposure would require a header change.

---

### S4 — Backslash-escaped wikilinks in table cells are valid Obsidian Markdown but confuse plain-text parsers

Multiple pages use `[[components/foo\|bar]]` syntax (backslash before pipe) inside Markdown table cells. This is the standard Obsidian escaping convention for pipes inside table cells and is functionally correct. However, plain-text lint tools see these as targets ending in `\` and report false dead-links. These are not real dead links — all targets in the `\|` pattern resolve correctly. No fix needed for vault health; document as a known false-positive class if running external linters.

Affected files include: `wiki/components/compat.md`, `wiki/components/api.md`, `wiki/dependencies/_index.md`, `wiki/flows/_index.md`, `wiki/components/parser.md`, `wiki/components/xasl.md`, `wiki/components/parallel-query.md`, `wiki/flows/dml-execution-path.md`, and others.

---

### S5 — `wiki/Architecture Overview.md` links `[[modules/cs]]` which does not exist

`wiki/Architecture Overview.md` contains a link to `[[modules/cs]]`. No file `wiki/modules/cs.md` exists. The same dead link appears in `wiki/components/csql-shell.md`, `wiki/concepts/Build Modes (SERVER SA CS).md`, `wiki/components/utility-binaries.md`. A `modules/sa` and `modules/cubrid` page also appear as dead links in those files.

Fix: either create stub pages `wiki/modules/cs.md`, `wiki/modules/sa.md`, `wiki/modules/cubrid.md`, or redirect the links to `wiki/modules/src.md` which covers the full source tree.

---

### S6 — `wiki/components/query-regex.md` contains malformed wikilinks from regex pattern text

`wiki/components/query-regex.md` contains `[[. .]]`, `[[.x.]]`, `[[:alpha:]]` which are not wikilinks — they are regex examples that happen to be double-bracket-delimited. These are false positives for dead-link checkers.

Fix: escape or reformat the regex examples as code spans (`` `. .` `` etc.) so they are not parsed as wikilinks.

---

### S7 — `wiki/entities/Claude SEO.md` links `[[E-commerce SEO]]` which does not exist

`wiki/entities/Claude SEO.md` is a pre-CUBRID legacy entity page (not moved to `_legacy/`). It links to `[[E-commerce SEO]]` which has no corresponding page anywhere in the vault.

Fix: either move this page to `wiki/_legacy/entities/` (consistent with the other legacy entities), or remove the dead link if keeping it.

---

### S8 — `wiki/components/cm-common-src.md` links `[[modules/cm_common]]` which does not exist

No `wiki/modules/cm_common.md` file exists.

Fix: create a stub `wiki/modules/cm_common.md` or redirect to `wiki/modules/src.md`.

---

### S9 — `wiki/components/cub-server-main.md` links `[[modules/cubrid]]` which does not exist

No `wiki/modules/cubrid.md` file exists. Same issue as S5 — the `cubrid` binary's module page was never created.

Fix: create `wiki/modules/cubrid.md` stub, or consolidate with `wiki/modules/src.md`.

---

### S10 — `wiki/components/win-tools.md` links `[[modules/win]]` which does not exist

No `wiki/modules/win.md` exists.

Fix: create a stub or remove the link.

---

### S11 — `wiki/dependencies/_index.md` links `[[cubrid-cci]]`, `[[cubrid-jdbc]]`, `[[cubridmanager]]` without path prefix

Three links at the bottom of `wiki/dependencies/_index.md` use bare names (`[[cubrid-cci]]`, `[[cubrid-jdbc]]`, `[[cubridmanager]]`). The actual pages are at `wiki/modules/cubrid-cci.md`, `wiki/modules/cubrid-jdbc.md`, `wiki/modules/cubridmanager.md`. Obsidian resolves these by filename uniqueness; lint tools do not.

Fix: change to `[[modules/cubrid-cci]]`, `[[modules/cubrid-jdbc]]`, `[[modules/cubridmanager]]`.

---

### S12 — Legacy `wiki/entities/Claude SEO.md` not moved to `_legacy/`

The 2026-04-24 lint pass moved 18 pre-CUBRID pages to `wiki/_legacy/`. `wiki/entities/Claude SEO.md` appears to have been missed — it has no CUBRID content.

Fix: move to `wiki/_legacy/entities/Claude SEO.md` and update `wiki/entities/_index.md` accordingly.

---

## Baseline Staleness

### Authoritative baseline: `05a7befd8b714811632a16a97d3683ab3b397a0f`

Files checked for stale `0be6cdf6` references:

| File | Verdict |
|---|---|
| `wiki/hot.md` | OK — references are historical `[!update]` callouts documenting the chain of bumps. Expected. |
| `wiki/components/parallel-hash-join.md` | OK — `[!update]` callout cites PR #6981 merge commit `0be6cdf6` as the merge SHA. This is accurate: `0be6cdf6` was the merge commit of PR #6981, not the "current baseline" claim. |
| `wiki/components/file-manager.md` | OK — same pattern, `0be6cdf6` is the PR #6981 merge SHA in a `[!update]` callout. |
| `wiki/components/list-file.md` | OK — `[!update]` callout, same pattern. |
| `wiki/components/parallel-hash-join-task-manager.md` | OK — `[!update]` callout, same pattern. |
| `wiki/components/parallel-index-scan.md` | OK — states "does **not** exist in baseline `0be6cdf6`" which is accurate historical context for a branch-WIP page. |
| `wiki/components/parallel-list-scan.md` | OK — same historical context statement. |
| `wiki/flows/parallel-list-scan-open.md` | OK — same historical context statement. |
| `wiki/sources/2026-04-28-tfile-role-analysis.md` | OK — multiple references are all in historical context ("captured on the day baseline was bumped to `0be6cdf6`", "verified against baseline `0be6cdf6`"). These are accurate statements about when the analysis was done. |
| `wiki/index.md` | **STALE** — line 32: `Baseline: \`65d69154\`` (two bumps behind). See C2 above. |

No component or concept page outside the PR/branch-WIP pages uses `0be6cdf6` as a claimed authoritative baseline. The index.md staleness (C2) is the only actionable baseline-staleness issue.

---

## Address Validation (DragonScale Mechanism 2)

- Counter state: `8` (next allocation = `c-000008`)
- Highest c- address observed: `c-000007`
- Post-rollout pages checked: 7 (5 passing, **2 errors** — see C3)
- Address-map consistency: 6 entries checked, all pass (frontmatter matches manifest)
- Legacy pages pending backfill: 254 (expected; no action required)
- Address gap at `c-000002`: intentional test-run consumption (logged); no page to create

### Errors
- `wiki/prs/PR-6930-lock-manager-init-refactor.md` — missing `address:`. Created 2026-05-08 (post-rollout). Run `./scripts/allocate-address.sh` and add the result to frontmatter and `.raw/.manifest.json`.
- `wiki/prs/PR-7102-db-get-char-intl-cleanup.md` — missing `address:`. Created 2026-05-08 (post-rollout). Same fix.

### Pending backfill (informational)
- 254 legacy pages (created before 2026-04-29) without addresses. Backfill is optional per spec.

---

## Semantic Tiling (DragonScale Mechanism 3)

- `scripts/tiling-check.py --peek` exit code: **1** (unexpected — script runtime error)
- Error: `TypeError: 'type' object is not subscriptable` at `parse_frontmatter` annotation `tuple[dict, str]` (PEP 585, requires Python 3.9+; system has Python 3.6.8)
- Tiling status: **disabled** (not exit 10/11 — this is a script compatibility bug, not an ollama availability issue)
- No tiling report generated

See C4 above for fix options.

---

## Index Integrity

- `wiki/prs/_index.md`: both PR-6930 and PR-7102 are listed with full slugs, merge dates, case classification, and baseline-bump details. **Pass.**
- `wiki/log.md`: two new baseline-bump entries at the top (`5e12a293 → 05a7befd` for PR #7102, `0be6cdf6 → 5e12a293` for PR #6930), both dated 2026-05-08. **Pass.**
- `wiki/hot.md`: baseline commit updated to `05a7befd`, `[!update]` chain documents both bumps. **Pass.**
- `CLAUDE.md`: baseline hash updated to `05a7befd8b714811632a16a97d3683ab3b397a0f`. **Pass.**

---

## PR Wikilink Resolution (focus area)

All wikilinks in `wiki/prs/PR-6930-lock-manager-init-refactor.md` and `wiki/prs/PR-7102-db-get-char-intl-cleanup.md` resolve correctly:

| Link | Resolves |
|---|---|
| `[[components/lock-manager]]` | wiki/components/lock-manager.md — exists |
| `[[components/deadlock-detection]]` | wiki/components/deadlock-detection.md — exists |
| `[[components/db-value]]` | wiki/components/db-value.md — exists |
| `[[components/base]]` | wiki/components/base.md — exists |
| `[[components/cas]]` | wiki/components/cas.md — exists |
| `[[components/loaddb-executor]]` | wiki/components/loaddb-executor.md — exists |
| `[[components/authenticate]]` | wiki/components/authenticate.md — exists |
| `[[components/csql-shell]]` | wiki/components/csql-shell.md — exists |
| `[[components/loaddb]]` | wiki/components/loaddb.md — exists |
| `[[sources/cubrid-src-transaction]]` | wiki/sources/cubrid-src-transaction.md — exists |
| `[[sources/cubrid-src-base]]` | wiki/sources/cubrid-src-base.md — exists |
| `[[sources/cubrid-src-broker]]` | wiki/sources/cubrid-src-broker.md — exists |
| `[[sources/cubrid-src-compat]]` | wiki/sources/cubrid-src-compat.md — exists |
| `[[prs/_index]]` | wiki/prs/_index.md — exists |
| `[[log]]` | wiki/log.md — exists |

No dead wikilinks in either new PR page.

All wikilinks in the six recently-edited component pages that were introduced by the PR ingest also resolve correctly. The backslash-pipe (`\|`) links in table cells (S4) are Obsidian escaping conventions, not dead links.

---

## Orphan Check (focus area)

All recently-added and recently-edited pages are linked from at least one other page:
- `prs/PR-6930-lock-manager-init-refactor` — linked from `prs/_index.md`, `log.md`, `hot.md`, `components/lock-manager.md`, `components/deadlock-detection.md`
- `prs/PR-7102-db-get-char-intl-cleanup` — linked from `prs/_index.md`, `log.md`, `hot.md`, `components/db-value.md`, `components/base.md`, `components/cas.md`, `components/loaddb-executor.md`
- All six reconciled component pages are linked from `components/_index.md` and cross-linked from related pages.

No orphans in the focus area.

---

## Recommended Fix Order

1. **Now** — C1: bump `updated: 2026-05-08` on 6 component pages (6 one-line edits)
2. **Now** — C2: update `wiki/index.md` line 32 baseline hash and date (1 line edit)
3. **Now** — C3: allocate addresses for PR-6930 and PR-7102 pages and register in manifest (2 allocations)
4. **Soon** — C4: fix `tiling-check.py` Python 3.6 compatibility (add `from __future__ import annotations`)
5. **Soon** — W3: add `status: branch-wip` to `sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables.md`
6. **Batch** — W1+W2+W4+W7: bulk frontmatter gap fixes across sources/ and component pages
7. **Low priority** — S5/S8/S9/S10: create module stubs for `cs`, `sa`, `cubrid`, `win`, `cm_common`
8. **Low priority** — S6: reformat regex examples in `query-regex.md` as code spans
9. **Low priority** — S12: move `entities/Claude SEO.md` to `_legacy/`
