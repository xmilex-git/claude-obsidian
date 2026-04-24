---
type: meta
title: "Wiki Lint Report — 2026-04-24"
created: 2026-04-24
updated: 2026-04-24
tags:
  - meta
  - lint
  - health
status: evergreen
related:
  - "[[index]]"
  - "[[log]]"
  - "[[lint-report-2026-04-23]]"
---

# Wiki Lint Report — 2026-04-24

Vault root: `/home/cubrid/claude-obsidian`
Scope: full vault (Mode B CUBRID + legacy seed)
Prior report: [[lint-report-2026-04-23]]

## Summary

- **Pages scanned:** 246 (was 209, **+37** since prior lint — all from late 2026-04-23 round-5 commits)
- **Unique wikilink targets:** 263
- **Issues found:** 31 distinct (fewer than prior because systemic categories are now tracked as 3–4 clusters, not per-link)

| Category | Prior | Now | Δ |
|---|---:|---:|:---:|
| Dead wikilinks (unique targets, de-duped) | 44 | **21** | ▼ 23 |
| Orphan pages | 1 | **2** | ▲ 1 |
| Frontmatter gaps — missing `created` | — | **52** | — |
| Frontmatter gaps — missing `updated` | — | **35** | — |
| Frontmatter gaps — missing `status` | — | **24** | — |
| Empty section headings | 72 pages | *not re-scanned* | — |
| Confirmed stale claims | 2 | **1** | ▼ 1 |
| Status mismatches | 1 | **1** | = |
| Duplicate basenames | — | **1** cluster (`_index.md` ×8) | — |

---

## Delta vs 2026-04-23

### Fixed ✅

- **C-4 (underscore typos in `modules/src.md`)** — verified: `components/cm_common-src` and `components/win_tools` references are gone.
- **C-6 (`wait_for_graph.c` misleading in `components/transaction.md`)** — verified: line 10 frontmatter now reads `wait_for_graph.c / wait_for_graph.h (legacy — gated by ENABLE_UNUSED_FUNCTION; active detector is in lock_manager.c)`; line 95 table cell now qualifies both files; lines 191 and 205 add the explicit gate note.

### Partially addressed 🟡

- **S-4 (`overview.md` stale content)** — counts updated (`Sources ingested: 32`, realistic), but the two legacy dead links (`[[AI Marketing Hub Cover Images Canvas]]`, `[[claude-obsidian-presentation]]`) are still on the page — now labelled "legacy seed" rather than removed. Links still fail resolution in the graph.
- **C-7 (`strict_warnings` ambiguity in `components/_index.md`)** — line 232 still reads `strict_warnings (referenced, not yet in tree)`. Parenthetical hedge present but not emphatic. Keeping this flagged at lower severity.

### Not fixed (carry forward from 04-23) 🔴

| Prior ID | Item | Current status |
|---|---|---|
| C-1 | Driver submodule stubs (`cubrid-cci`, `cubrid-jdbc`, `cubridmanager`) | 3 targets still dead |
| C-2 | CUBRID module stubs (`cs`, `sa`, `cubrid`, `conf`, `win`, `cm_common`) | 6 targets still dead |
| C-3 | `modules/_index` bare-name links (`cmake`, `debian`, `demo`, `docs`, `include`, `util`, + overlap with C-2) | 11 bare-name dead targets |
| C-5 | `[[Wiki Map]]` dead link | 4 references remain: `index.md:14`, `index.md:31`, `getting-started.md:14`, `getting-started.md:91`, `concepts/_index.md:14` |
| W-1 | `meta/claude-obsidian-v1.4-release-session` orphan | still plain-text path in `log.md:129`, zero inbound wikilinks |
| W-2 | `cherry-picks#N` anchor links broken | 14 links broken across 5 entity pages (`rvk7895`, `kepano`, `ballred`, `Ar9av`, `Nexus`) |
| W-3 | `overview.md` legacy seed dead links | see "Partially addressed" above |
| W-5 | `[[wikilinks]]` bare link in `concepts/cherry-picks` | still present |
| W-10 | `decisions/_index.md` status mismatch | still declared `status: active` despite stub content (only `_index.md` in the folder) |
| S-5 | `replication` / HA page missing | no page created; no new references filed |
| S-8 | `flows/` pages orphaned from `Data Flow` hub | unchanged |

### New since 04-23 🆕

- **Orphan page — `components/query-reevaluation.md`** (9,597 bytes). Exists on disk but zero inbound wikilinks. Likely a round-5 ingest artifact that was filed without being cross-referenced from its parent component page (probably `components/scan-manager` or `components/query-executor`).
- **37 new pages** added in round-5 commits on 2026-04-23 (after the prior lint ran). Net content growth is healthy; most of the new pages are `sources/cubrid-src-*` (correctly indexed in `sources/_index.md`) and `components/parallel-*` (correctly cross-linked from `components/_index.md` and `sources/cubrid-src-query-parallel.md`).
- **Round-5 defects DID land as inline notes** (not as dedicated pages, which is fine):
  - `sort_copy_sort_param` missing impl → captured in `components/parallel-sort.md`
  - `TASK_QUEUE_SIZE_PER_CORE = 2` unused → captured in `components/parallel-worker-manager-global.md`
  - `reset_queue` epoch-bump invariant → captured in `components/parallel-task-queue.md`

---

## Active Issues

### Dead Wikilinks (21 unique broken targets)

De-duped, false positives removed. Grouped by cause:

**Driver / submodule module stubs (3 — C-1):** `cubrid-cci`, `cubrid-jdbc`, `cubridmanager`

**CUBRID build dirs not yet ingested (6 — C-2):** `cs`, `sa`, `cubrid`, `conf`, `win`, `cm_common`

**`modules/_index` bare-name links (6 additional, overlap with C-2/C-3):** `cmake`, `debian`, `demo`, `docs`, `include`, `util`

**Legacy seed dead links (3):** `Wiki Map`, `claude-obsidian-presentation`, `AI Marketing Hub Cover Images Canvas`

**Naming inconsistency (2):** `cm_common-src` (page is `cm-common-src`), `win_tools` (page is `win-tools`) — these may be prior typos or stale references surviving a rename.

**Small one-offs (1):** `wikilinks` bare link in `concepts/cherry-picks.md` (W-5).

**False positives filtered out:** `. .`, `.x.`, `:alpha:`, `name` (regex samples inside code fences), `dashboard.base` (Obsidian Bases file — resolves natively in Obsidian 1.9+).

---

### Orphan Pages (2)

- `lint-report-2026-04-23` — **expected orphan** (lint reports are never cross-linked; this one does appear in this report's `related` field so the chain is maintained).
- `components/query-reevaluation` — **new unintentional orphan**. 9.6 KB of content with zero inbound wikilinks. Suggested fix: add a link from `components/scan-manager` (the most likely parent) or from `concepts/Query Processing Pipeline`. Candidate link anchor: wherever re-evaluation during index scan or spilled hash-join rebuild is discussed.

---

### Frontmatter Gaps

Vault-wide count:

| Field | Pages missing |
|---|---:|
| `created` | **52** |
| `updated` | **35** |
| `status` | **24** |
| `type` | 0 |
| `tags` | 0 |

**Clustered sources of the gap:**

- **Hub pages** (`Architecture Overview`, `Tech Stack`, `Data Flow`, `Dependency Graph`, `Key Decisions`) — all missing `created`. All have `status`, `updated`, `type`, `tags`.
- **`dependencies/` pages** (10) — all missing `status`. Confirmed via per-file scan: `flex-bison`, `jansson`, `libedit`, `libexpat`, `libtbb`, `lz4`, `openssl`, `rapidjson`, `re2`, `unixodbc`.
- **`sources/` pages** — majority missing `created`; several missing `status` (e.g. `cubrid-3rdparty`, `cubrid-src-base`, `cubrid-src-compat`, `cubrid-src-parser`, `cubrid-src-xasl`).
- **Index/root meta** (`index`, `hot`, `log`, `overview` — partial) — unchanged from prior lint.

Suggested batch: one awk/yq pass that back-fills `created: 2026-04-23` on all sources missing the field, and `status: reference` on all dependency pages.

---

### Stale Claims / Contradictions

- **1 remaining**: `strict_warnings` ambiguity in `components/_index.md:232` (C-7 partial).
- **Contradictions now correctly captured in multiple places** with explicit `(legacy — ENABLE_UNUSED_FUNCTION)` qualifiers (`wait_for_graph.c` is no longer a concern).

The hot cache lists both contradictions (`wait_for_graph.c`, `strict_warnings`) as still-open items — the hot cache itself is slightly stale on `wait_for_graph.c`. Minor; self-corrects on next rewrite.

---

### Status Mismatches

- `decisions/_index.md` — declared `status: active` but the directory contains only `_index.md` itself. No ADRs filed. Should be `status: stub` per prior lint W-10. **Still unfixed.**

---

### Duplicate Basenames

- `_index.md` appears 8 times (one per top-level section: `components/`, `concepts/`, `decisions/`, `dependencies/`, `entities/`, `flows/`, `modules/`, `sources/`).
- **Not a bug** — Obsidian resolves these correctly when wikilinks use the `folder/_index` qualified form (which the vault does consistently). Flagging only so future contributors know to always use the qualified form.

---

### Naming Conventions

No new violations. The dual convention documented in prior S-7 (Title Case for hubs/concepts, kebab-case for components/modules/sources/dependencies) continues to be applied consistently.

---

### Dashboard / Canvas

- `wiki/meta/dashboard.md` — **exists**, 1 content file + the companion `dashboard.base` Obsidian Base. Not re-scanned for query freshness this run.
- `wiki/meta/overview.canvas` — **does not exist**. Candidate to build; low priority.
- `wiki/Wiki Map.canvas` — **exists** at repo root (not under `meta/`). This is the actual canvas; the repeated `[[Wiki Map]]` dead wikilink (C-5) is because Obsidian wikilinks do not resolve to `.canvas` files by default. Resolution: either (a) change the four dead references from `[[Wiki Map]]` to `[[Wiki Map.canvas]]` (Obsidian-native canvas link syntax), or (b) create a thin `wiki/Wiki Map.md` companion that embeds the canvas with `![[Wiki Map.canvas]]`.

---

## Vault Health Assessment

**Improved** since 2026-04-23:
- Two confirmed stale claims reduced to one partial.
- Two critical dead-link clusters (C-4, underscore typos; C-6, `wait_for_graph.c` narrative) fully resolved.
- 37 new pages from round-5 are well-integrated — source pages indexed, component pages cross-linked, round-5 defect observations captured as inline notes rather than lost.

**Still outstanding** (editorial, not structural):
- Same 21 dead wikilinks, same orphan `v1.4-release-session`, same status mismatch on `decisions/_index`. These are known items deferred rather than regressions.
- One new unintentional orphan (`query-reevaluation`) — small, 1-line fix.

**Immediate priority actions (top 5, ranked by impact × effort):**

1. **Fix `[[Wiki Map]]` dead link (C-5)** — 4 references, single-pattern edit. Decide canvas-link vs. MD-stub approach first. *Highest impact; this is the visible broken link on the front-page `index.md`.*
2. **Cross-link `components/query-reevaluation`** — one-line edit in the relevant parent component page. Removes a new orphan.
3. **Batch frontmatter fix** — add `status: reference` to 10 `dependencies/` pages and `created: 2026-04-23` to ~50 source/hub pages. Single scripted pass.
4. **Demote `decisions/_index.md` to `status: stub`** — one-line edit.
5. **Remove or replace the two legacy-seed dead links in `overview.md`** (W-3 partial) — converts the page from "mostly true" to "fully consistent".

Deferred (same priority as prior lint): the three driver-submodule stubs (`cubrid-cci`, `cubrid-jdbc`, `cubridmanager`) and the six CUBRID build-dir stubs. These resolve naturally as ingest coverage extends.
