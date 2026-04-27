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
- **New issues introduced**: 5 (all minor — `[[hot.md]]` typos in my new manual pages)
- **Pre-existing issues remaining**: 3 categories — modules/_index dead links (14), missing `created` frontmatter (70), missing `tags` (16)
- **Auto-fixable**: yes for all 5 new + the modules/_index dead links (after deciding policy)

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

| Field | Pages missing | Delta vs 04-24 |
|---|---|---|
| `type` | **0** | -3 (was 3) ✅ |
| `status` | **6** | -18 (was 24) ✅ |
| `created` | **70** | +24 (was 46) ⚠️ — regression |
| `tags` | **16** | +N — new gap |
| **Any field missing** | **70 unique pages** | (mostly the same ~70 missing `created`) |

### Worst offenders (15 of 70 with missing `created`)

- `wiki/modules/contrib.md`
- `wiki/prs/PR-6911-parallel-heap-scan-io-bottleneck.md`
- `wiki/prs/PR-7062-parallel-scan-all-types.md`
- `wiki/prs/PR-6753-optimizer-histogram-support.md`
- `wiki/prs/PR-6689-bcb-mutex-to-atomic-latch.md`
- `wiki/prs/PR-6443-system-catalog-information-schema.md`
- `wiki/prs/PR-7011-parallel-index-build.md`
- `wiki/components/aggregate-analytic.md`
- `wiki/components/authenticate.md`
- `wiki/components/base.md`
- `wiki/components/broker-impl.md`
- `wiki/components/btree.md`
- `wiki/components/cas.md`
- `wiki/components/cm-common-src.md`
- `wiki/components/communication.md`

> [!gap] Why is `created` regression-up?
> The 04-24 lint added `created: 2026-04-23` to 46 pages. Many component pages still don't have it — the prior fix was incomplete. PR pages added since are a new gap (PR ingest workflow doesn't set `created`).

### `status` gaps (6 pages — sample)
Need to check individual pages; previous report fixed 24 of 30. Likely `status: stub` would be appropriate for any page < 50 lines without strong content.

### `tags` gaps (16 pages)
Mostly old hub pages or scaffolding. Less critical than `created` since type-driven discovery still works.

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

## Recommended Top-5 Fixes

1. **Fix 5 `[[hot.md]]` → `[[hot]]`** in the 3 new manual pages I authored today. Trivial sed across 3 files. (5 minutes.)
2. **Resolve `wiki/modules/_index.md` dead links** (14 targets). Recommend the **demote** approach: convert dead `[[name]]` to plain text + `> [!gap]` callout per missing page; keep `[[cubrid-cci]]`/`[[cubrid-jdbc]]`/`[[cubridmanager]]` as live links and create stub module pages so cross-references from `Architecture Overview` and `Tech Stack` work. (20 minutes.)
3. **Add `created: 2026-04-23` to 70 pages missing it** — sed batch (matches 04-24's same fix, but covers pages it missed). Most are component pages and PR pages. (10 minutes for sed; verify mtime alignment.)
4. **Add `tags: [component, cubrid]` (or appropriate) to 16 pages missing tags** — manual or scripted. (15 minutes.)
5. **Set `status:` on the 6 pages missing it** — likely `status: developing` or `status: stub` based on length. (10 minutes.)

Total estimated work: **~60 minutes** to bring the wiki to lint-clean.

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
