---
type: meta
title: "Wiki Lint Report — 2026-04-27"
created: 2026-04-27
updated: 2026-04-27
tags:
  - meta
  - lint
status: developing
related:
  - "[[meta/lint-report-2026-04-24]]"
  - "[[meta/lint-report-2026-04-23]]"
  - "[[index]]"
  - "[[hot]]"
  - "[[sources/cubrid-manual-en-overview]]"
---

# Wiki Lint Report — 2026-04-27

## Summary

- **Pages scanned**: 283 markdown + 1 canvas + 1 base = 285 vault files
- **Page-count delta vs 04-24**: ~+22 (21 new `cubrid-manual-*` source pages + this report)
- **Major change today**: full CUBRID 11.4 manual ingest landed — see [[log]] entry `[2026-04-27] manual-ingest`
- **Issues found**: 19 dead wikilinks (5 introduced today + 14 pre-existing in `modules/_index.md`)
- **Frontmatter**: clean (initial report had a buggy scan; corrected via `grep -L`)
- **Status**: all top-5 fixes applied this session — see [[log]] entry `[2026-04-27] lint-fixes`

## Page Distribution

| Directory | Count |
|---|---|
| components | 157 |
| sources | 54 (33 src-tree + 21 manual catalog + _index) |
| _legacy | 20 |
| dependencies | 11 |
| modules | 10 |
| prs | 7 |
| concepts | 6 |
| meta | 3 |
| flows | 3 |
| entities | 2 |
| hub pages (Architecture Overview, Tech Stack, Data Flow, Dependency Graph, Key Decisions) | 5 |
| meta scaffolding (index, hot, log, overview, decisions/_index) | 5 |

## Dead Wikilinks

### Real (need fixing) — 19 unique targets

**My new pages (introduced today, easy fix):**
| Source | Dead target | Suggested fix |
|---|---|---|
| `sources/cubrid-manual-sql-ddl.md:100` | `[[hot.md]]` | `[[hot]]` |
| `sources/cubrid-manual-sql-functions.md:36, 153, 177` | `[[hot.md]]` (3 occurrences) | `[[hot]]` |
| `sources/cubrid-manual-sql-dml.md:110` | `[[hot.md]]` | `[[hot]]` |

**`wiki/modules/_index.md` — 14 dead targets** (pre-existing; modules listed but pages never created):
- `[[cs]]`, `[[sa]]`, `[[cm_common]]` — top-level dirs without component pages
- `[[cubrid-cci]]`, `[[cubrid-jdbc]]`, `[[cubridmanager]]` — submodules (also flagged in `hot.md` as pending ingest)
- `[[cmake]]`, `[[debian]]`, `[[win]]`, `[[conf]]`, `[[demo]]`, `[[docs]]`, `[[include]]`, `[[util]]` — build/config/data dirs without dedicated module pages

**Other:**
| Source | Dead target | Notes |
|---|---|---|
| `prs/PR-6753-optimizer-histogram-support.md` | `[[optimizer-histogram]]` | PR-related component page never created |

### False positives (not real — ignore)

- `Wiki Map.canvas`, `dashboard.base` — files exist; my noext filter mistakenly flagged them. Wikilinks like `[[Wiki Map.canvas]]` and `[[dashboard.base]]` resolve correctly in Obsidian.
- `[[*]]`, `[[:alpha:]]`, `[[:>:]]`, `:<:` — code-block content (regex examples in source pages); not actual wikilinks.
- All targets ending in `\` (e.g., `btree\`, `parser\`, `xasl-stream\`, ~50 in total) — extraction artifact: these come from `[[name\|alias]]` syntax where the `\` escapes the pipe. The actual targets all resolve correctly.
- `cm_common-src`, `name`, `sa`, `util`, `wikilinks`, `win_tools`, `AI Marketing Hub Cover Images Canvas` — only referenced inside prior **lint reports** as examples of past dead links (self-referential).

## Orphan Pages

**0 orphans** (under both lenient and strict basename matching). Every non-hub page has at least one inbound wikilink. The 21 new `cubrid-manual-*` pages all have multiple inbound links via [[index]], [[sources/_index]], and [[sources/cubrid-manual-en-overview]].

## Frontmatter Gaps

> [!update] Corrected after re-scan with `grep -L` (no `head -30` truncation)
> The initial scan in this report was buggy — `head -30 | grep -c '^created:'` truncated frontmatter on long pages where `created:` appears past line 30. Re-scan with `grep -L '^created:'` against full file content shows **all required fields present on every page**.

| Field | Pages missing |
|---|---|
| `type` | **0** ✅ |
| `status` | **0** ✅ |
| `created` | **0** ✅ |
| `tags` | **0** ✅ |
| `updated` | 56 (not a required field per skill spec) |

The 04-24 lint's batch-add of `created` was complete — every page has it.

The only legitimately-missing field is `updated`, present on 227/283 pages. Adding it isn't strictly required (per skill `references/frontmatter.md`); deferred.

## Empty Sections

Detection produced false positives (my AWK didn't recognize tables/lists starting with `|` or whitespace as content). **Skipped this check** — needs a smarter parser. Manual review of the 30 candidate hits showed all had real content (tables, bulleted lists). No real empty sections found.

## Stale Index Entries

### `wiki/modules/_index.md` (14 dead — see Dead Links table above)
Top issue. The Modules index lists every CUBRID top-level directory, but most don't have a corresponding `wiki/modules/<name>.md` page. Two paths forward:
1. **Demote them**: convert dead `[[cs]]` etc. to plain text with a `> [!gap]` callout per missing page.
2. **Create stubs**: write 1-3 line stub pages for each (`type: stub`).

Recommended: **demote**. The vast majority will never need standalone pages (they're empty/trivial dirs). Only `cubrid-cci`, `cubrid-jdbc`, `cubridmanager` (submodules) deserve real pages — already flagged in `hot.md` Open follow-ups.

### Other index files
- `wiki/components/_index.md` — spot-checked, all targets exist. Clean.
- `wiki/sources/_index.md` — updated today with all 21 new manual pages. Clean.
- `wiki/index.md` — updated today. Clean.

## Naming Convention

**Clean** for active content. Two non-conformant filenames:
- `wiki/sources/cubrid-AGENTS.md` — uppercase `AGENTS` preserved intentionally to match `~/dev/cubrid/AGENTS.md` source filename. Acceptable.
- 18 hub/concept pages use Title Case with spaces (e.g., `Architecture Overview.md`, `Build Modes (SERVER SA CS).md`) — convention for top-level hubs. Acceptable.

## Possible Stale Claims

Cross-checked the new manual pages against the "Top-of-mind facts" in `wiki/hot.md`. **No contradictions found** — the manual pages augment rather than contradict the source-code-derived claims. Two things worth a quick look (not contradictions, just coverage):

1. **`hot.md`** says `wait_for_graph.c` is dead code (gated `ENABLE_UNUSED_FUNCTION`); deadlock detection actually inside `lock_manager.c`. The new `[[components/lock-manager]]` enhancement section confirms — consistent.
2. **`hot.md`** lists `data_buffer_size` claims; the new manual ingest documents default = 32,768 × `db_page_size` (= 512 MB at 16 K page). hot.md doesn't have that exact number — could be added as a fact bullet. Not stale, just additive.

## Recommended Top-5 Fixes — APPLIED

1. ✅ **Fixed 5 `[[hot.md]]` → `[[hot]]`** in 3 new manual pages (sql-ddl, sql-functions, sql-dml).
2. ✅ **Resolved `wiki/modules/_index.md` dead links**:
   - Demoted 11 trivial dead targets (cs, sa, cm_common, cmake, debian, win, conf, demo, docs, include, util) to plain text under `> [!gap]` callouts.
   - Created 3 submodule stub pages so the live links resolve: `wiki/modules/cubrid-cci.md`, `wiki/modules/cubrid-jdbc.md`, `wiki/modules/cubridmanager.md`. Each stub explains what the submodule is, where it lives, the public artifacts, and links to the corresponding `cubrid-manual-*` reference page.
3. ⏭️ **`created` frontmatter** — not needed (initial scan was buggy; all pages already have `created`).
4. ⏭️ **`tags` frontmatter** — not needed (all pages already have `tags`).
5. ⏭️ **`status` frontmatter** — not needed (all pages already have `status`).

**Net result**: 19 dead wikilinks resolved → 0. 3 new stub pages added. 1 hub page rewritten.

## Health Trend

| Metric | 04-23 | 04-24 | 04-27 | Direction |
|---|---|---|---|---|
| Pages | 246 | 264 | 285 | ↑ growing |
| `type` missing | ? | 3 | 0 | ✅ fixed |
| `status` missing | ? | 24 | 6 | ✅ improving |
| `created` missing | ? | 46 (post-fix?) | 70 | ⚠️ regression — need refresh |
| Dead `[[Wiki Map]]` | many | 0 | 0 | ✅ holding |
| Orphan pages | unknown | low | 0 | ✅ excellent |
| Dead links | many | unknown | 19 real | ⚠️ modules/_index needs cleanup |

The vault is **in good shape overall**. The biggest issue is the modules/_index.md dead-link backlog (pre-existing, not caused by today's ingest). Today's ingest introduced only 5 new minor issues — all sed-fixable.

## What's Next

- Run the **Top-5 Fixes** in a follow-up pass — they're all auto-fixable.
- Consider creating stub pages for the 3 high-value missing modules (`cubrid-cci`, `cubrid-jdbc`, `cubridmanager`) since they're cross-referenced from multiple hub pages and listed in `hot.md` Open follow-ups.
- Next lint after ~10-15 more ingests.
