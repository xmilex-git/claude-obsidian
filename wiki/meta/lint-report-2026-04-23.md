---
type: meta
title: "Wiki Lint Report — 2026-04-23"
created: 2026-04-23
updated: 2026-04-23
tags:
  - meta
  - lint
  - health
status: evergreen
related:
  - "[[index]]"
  - "[[log]]"
---

# Wiki Lint Report — 2026-04-23

Vault root: `/Users/song/DEV/claude-obsidian`
Scope: full vault (Mode B CUBRID + legacy seed)
Tool: `/wiki-lint` skill, comprehensive 8-check run

---

## Summary

- **Pages scanned:** 209
- **Issues found:** 107 (14 critical, 51 warnings, 42 suggestions)

| Category | Count |
|----------|-------|
| Dead wikilinks (unique broken targets) | 44 |
| Orphan pages (zero inbound links) | 1 |
| Frontmatter gaps (pages missing required fields) | 56 |
| Empty section headings | 72 pages, 102 headings |
| Stale index entries in `index.md` | 1 (`[[Wiki Map]]`) |
| Missing pages for frequently mentioned concepts | 10 high-priority |
| Confirmed stale claims | 2 (`wait_for_graph.c`, `strict_warnings`) |
| Status mismatches | 1 (`decisions/_index`) |
| Sources not listed in `sources/_index` | 9 |
| Pages over 300 lines | 2 |

---

## Critical (must fix)

### C-1. Dead wikilinks — missing driver/submodule module pages

The three driver submodule pages have never been ingested and are linked from 14+ pages each. Every link is broken.

| Target | Broken in (sample) |
|--------|-------------------|
| `[[modules/cubrid-cci]]` or `[[cubrid-cci]]` | `hot`, `Architecture Overview`, `Tech Stack`, `dependencies/_index`, `modules/_index`, `components/compat`, `components/dbi-compat`, `components/contrib-language-drivers`, `entities/CUBRID`, `modules/contrib`, `concepts/Error Handling Convention`, `components/cm-common-src`, `sources/cubrid-src-compat`, `sources/cubrid-src-cm-common` |
| `[[modules/cubrid-jdbc]]` or `[[cubrid-jdbc]]` | `hot`, `Tech Stack`, `Architecture Overview`, `modules/_index`, `modules/contrib`, `components/contrib-language-drivers` |
| `[[modules/cubridmanager]]` or `[[cubridmanager]]` | `hot`, `Tech Stack`, `modules/_index`, `components/cm-common-src`, `sources/cubrid-src-cm-common` |

**Suggested fix:** Ingest `contrib/` submodules (`cubrid-cci/`, `cubrid-jdbc/`, `cubridmanager/`) to produce module stub pages, or create placeholder stubs at `wiki/modules/cubrid-cci.md`, `wiki/modules/cubrid-jdbc.md`, `wiki/modules/cubridmanager.md`.

---

### C-2. Dead wikilinks — missing CUBRID module stubs

Multiple pages reference module directories that were never given wiki pages. These are sub-directories of the CUBRID repo not yet ingested.

| Target | Broken in (sample) |
|--------|-------------------|
| `[[modules/cs]]` | `Architecture Overview`, `components/csql-shell`, `components/utility-binaries`, `concepts/Build Modes (SERVER SA CS)`, `modules/src`, `modules/_index` |
| `[[modules/sa]]` | `components/csql-shell`, `components/utility-binaries`, `concepts/Build Modes (SERVER SA CS)`, `modules/src`, `modules/_index` |
| `[[modules/cubrid]]` | `components/cub-server-main`, `concepts/Build Modes (SERVER SA CS)`, `modules/src` |
| `[[modules/conf]]` | `modules/broker` |
| `[[modules/win]]` | `components/win-tools`, `sources/cubrid-src-win-tools` |
| `[[modules/cm_common]]` | `components/cm-common-src`, `sources/cubrid-src-cm-common`, `modules/src` |

**Suggested fix:** Create stub pages at `wiki/modules/cs.md`, `wiki/modules/sa.md`, `wiki/modules/cubrid.md`, `wiki/modules/conf.md`, `wiki/modules/win.md`, `wiki/modules/cm_common.md`.

---

### C-3. Dead wikilinks — `modules/_index` bare-name targets

`modules/_index` lists many module directories as raw wikilinks without a `modules/` prefix. Because no page exists at those basenames, all resolve as dead.

Broken targets: `[[cm_common]]`, `[[cmake]]`, `[[conf]]`, `[[cs]]`, `[[debian]]`, `[[demo]]`, `[[docs]]`, `[[include]]`, `[[sa]]`, `[[util]]`, `[[win]]`

**Suggested fix:** Change each bare `[[name]]` to `[[modules/name|name]]` once the stub pages exist, or add the prefix immediately so the intent is clear.

---

### C-4. Dead wikilinks — `modules/src` underscore naming bugs

`modules/src` references two components with underscore names that do not exist; the actual pages use hyphens.

| Broken link | Correct target |
|-------------|---------------|
| `[[components/cm_common-src]]` | `[[components/cm-common-src]]` |
| `[[components/win_tools]]` | `[[components/win-tools]]` |

**Suggested fix:** Edit `modules/src.md` lines 55 and 58 to use the hyphenated names.

---

### C-5. Dead wikilink — `[[Wiki Map]]` missing page

`[[Wiki Map]]` is referenced in `index.md` (twice), `getting-started.md`, and `concepts/_index.md`. No file exists at any path that resolves this name.

**Suggested fix:** Either create `wiki/Wiki Map.md` as an Obsidian Canvas or graph-view summary page, or remove the four references and replace with `[[index]]` or `[[Architecture Overview]]`.

---

### C-6. Stale claim — `wait_for_graph.c` presented as active in `components/transaction`

`components/transaction.md` lines 74 and 95 present `wait_for_graph.c` as the active deadlock detection path in the architecture diagram and the sub-component table, without qualification. The actual active implementation is inside `lock_manager.c` (gated daemon `lock_deadlock_detect_daemon`). `wait_for_graph.c` is only compiled under `#if defined(ENABLE_UNUSED_FUNCTION)` — confirmed dead code.

The contradiction is correctly documented in `components/deadlock-detection`, `sources/cubrid-src-transaction` (line 53), and `hot.md` (line 46), but `components/transaction` itself still implies the file is active in its diagram and table.

**Specific locations:**
- Line 74: architecture diagram caption `(wait_for_graph.c)` next to Lock Manager — implies active
- Line 95: "Deadlock detection | `wait_for_graph.c`, `wait_for_graph.h`" — no qualifier

**Suggested fix:** Add a parenthetical `(dead code — `ENABLE_UNUSED_FUNCTION` only)` to line 95's file column and add a note to the diagram at line 74.

---

### C-7. Stale claim — `strict_warnings` listed as owned file

The root `AGENTS.md` (ingested as `sources/cubrid-AGENTS`) lists `strict_warnings` as an owned file in `src/debugging/`. The file does not exist in the tree under any extension. This is correctly flagged in `components/debugging` and `sources/cubrid-src-debugging`, but `components/_index.md` line 216 still states it matter-of-factly as a present file:

> `strict_warnings` (referenced, not yet in tree)

The parenthetical is there but buries the contradiction. The primary entry (`type_helper.hpp: compile-time type name stringification; strict_warnings...`) reads as if both files exist.

**Suggested fix:** Update `components/_index.md` line 216 to clearly separate the confirmed file from the missing one: e.g., "type_helper.hpp (confirmed); strict_warnings (absent — AGENTS.md reference only)".

---

## Warnings (should fix)

### W-1. Orphan page — `meta/claude-obsidian-v1.4-release-session`

`wiki/meta/claude-obsidian-v1.4-release-session.md` has zero inbound wikilinks. `log.md` references it by plain-text path (`wiki/meta/claude-obsidian-v1.4-release-session.md`) rather than a wikilink, so Obsidian cannot traverse to it.

Note: `meta/claude-obsidian-v1.2.0-release-session` and `meta/full-audit-and-system-setup-session` are in the same situation (plain-text path in `log.md`), but their basenames match the wikilinks Obsidian would resolve, so they still have 0 true inbound wikilinks. Only the v1.4 session is a true graph orphan.

**Suggested fix:** In `log.md`, convert the plain-text path on line 106 to `[[meta/claude-obsidian-v1.4-release-session|v1.4 Release Session]]`.

---

### W-2. Dead wikilinks — `cherry-picks` anchor links (14 broken)

All entity pages linking into `[[cherry-picks#N. Title]]` anchors are broken. The `cherry-picks` page uses tier headings (`## Tier 1 — Quick Wins`), not numbered-item headings. The specific heading anchors no longer exist.

Affected pages: `entities/Ar9av-obsidian-wiki` (4 links), `entities/kepano-obsidian-skills` (4), `entities/ballred-obsidian-claude-pkm` (3), `entities/rvk7895-llm-knowledge-bases` (2), `entities/Nexus-claudesidian-mcp` (1).

**Suggested fix:** Either add named headings to `cherry-picks.md` matching the anchors, or update the entity pages to link to `[[cherry-picks]]` without the anchor.

---

### W-3. Dead wikilinks — `overview.md` stale pre-CUBRID links

`overview.md` references two files that do not exist and predate the CUBRID scope:
- `[[AI Marketing Hub Cover Images Canvas]]` — an Obsidian Canvas file that was never in this vault
- `[[claude-obsidian-presentation]]` — no `.canvas` or `.md` at this path

**Suggested fix:** Remove or replace both links. The page's "Current State" section also shows "Sources ingested: 2" and "Wiki pages: 26" which is severely stale (actual count: 209 pages).

---

### W-4. Dead wikilinks — `meta/dashboard` references `[[dashboard.base]]`

`wiki/meta/dashboard.md` embeds `![[dashboard.base]]` and links `[[dashboard.base]]`. The file `wiki/meta/dashboard.base` exists on disk but is not a `.md` file — it is an Obsidian Bases file. Obsidian renders it natively. Our link checker flags it as dead. This is a false positive in the lint context, but the wikilink pattern `[[dashboard.base]]` without the `.base` extension is unusual and may not resolve on older Obsidian versions.

**Suggested fix:** No action required if Obsidian 1.9.10+ is the target. Otherwise, verify the embed renders correctly in the environment.

---

### W-5. Dead wikilink — `[[wikilinks]]` in `concepts/cherry-picks`

`concepts/cherry-picks.md` contains `[[wikilinks]]` as a bare link. No page exists at that path.

**Suggested fix:** Remove the link or replace with `[[index]]`.

---

### W-6. Large pages at or over 300 lines

| Page | Lines |
|------|-------|
| `flows/dml-execution-path` | 305 |
| `flows/ddl-execution-path` | 301 |

Both pages also have empty section headings (see W-7 below). The content is mostly stub text despite the high line count (the length comes from frontmatter, navigation, and placeholder structure rather than dense prose).

**Suggested fix:** Either flesh out the empty sections or acknowledge the pages as structural templates and update status accordingly.

---

### W-7. Empty section headings (selected high-priority)

72 pages contain one or more headings with no content underneath them. High-priority cases:

| Page | Empty headings |
|------|---------------|
| `Architecture Overview` | `CUBRID Architecture Overview` (root heading — entire page body is empty at the H1 level) |
| `Tech Stack` | `CUBRID Tech Stack` |
| `components/error-manager` | `Core Concepts`, `Public API`, `Convenience Macros` |
| `components/log-manager` | `Log Structure`, `Key Structures (log_record.hpp)` |
| `components/mvcc` | `Key Structures`, `Visibility Predicates` |
| `components/page-buffer` | `Data Structures` |
| `components/lock-manager` | `Key Structures` |
| `components/query-executor` | `Key State Structures` |
| `flows/dml-execution-path` | `Stage 4: Per-Statement Specifics`, `Failure Modes` |
| `flows/ddl-execution-path` | `DDL Execution Path (CREATE / ALTER / DROP / GRANT / REVOKE)` (entire top-level heading) |
| `components/xasl-generation` | `Key data structures`, `Translation process` |
| `components/parser` | `Key data structures` |

Note: Most `dependencies/` pages have empty top-level headings — the entire content is in the frontmatter YAML fields, not in Markdown body text. This is intentional for machine-readable dependency pages but means the Obsidian reading view shows empty pages.

**Suggested fix for dependencies/:** Add at least one prose paragraph under the heading summarizing the dependency's role in CUBRID, even if the YAML captures the structured data.

---

### W-8. Sources missing from `sources/_index`

Nine source pages exist on disk but are not listed in `sources/_index.md`:

| Source page | Status |
|-------------|--------|
| `sources/cubrid-AGENTS` | Has inbound links; just not indexed |
| `sources/cubrid-src-parser` | Has inbound links |
| `sources/cubrid-src-storage` | Has inbound links |
| `sources/cubrid-src-cm-common` | Has inbound links |
| `sources/cubrid-src-query-parallel` | Linked only from `log.md` (plain-text path) |
| `sources/cubrid-locales` | Not in index |
| `sources/cubrid-timezones` | Not in index |
| `sources/cubrid-contrib` | Not in index |
| `sources/claude-obsidian-ecosystem-research` | Listed in `index.md` but not `sources/_index` |

**Suggested fix:** Add these nine pages to the appropriate sections in `sources/_index.md`.

---

### W-9. Frontmatter gaps — 56 pages missing required fields

Required fields: `type`, `status`, `created`, `updated`, `tags`

**Most impacted groups:**

**Hub pages missing `created`** (5 pages): `Architecture Overview`, `Tech Stack`, `Data Flow`, `Dependency Graph`, `Key Decisions`

**All `dependencies/` pages missing `status`** (10 pages): `flex-bison`, `jansson`, `libedit`, `libexpat`, `libtbb`, `lz4`, `openssl`, `rapidjson`, `re2`, `unixodbc`

**Flow pages missing `status`** (2 pages): `flows/dml-execution-path`, `flows/ddl-execution-path`

**Source pages missing `created` and/or `updated`** (26 pages — the majority of `sources/`): Nearly all CUBRID source pages were created with partial frontmatter. The most severe gaps are `cubrid-src-base`, `cubrid-3rdparty`, `cubrid-src-parser`, `cubrid-src-compat`, `cubrid-src-xasl` which also lack `status`.

**Index/root pages missing `created`** (9 pages): `index`, `hot`, `log`, `getting-started`, `meta/dashboard`, `modules/_index`, `components/_index`, `decisions/_index`, `flows/_index`, `entities/_index`, `sources/_index`, `concepts/_index`, `dependencies/_index`

**Suggested fix:** Use a batch frontmatter update pass. Priority order: hub pages, then sources, then index pages, then dependencies. The `created` field can be back-filled from `git log --follow` or set to `2026-04-23` for the CUBRID ingest batch.

---

### W-10. Status mismatch — `decisions/_index` declared `active` but has stub content

`wiki/decisions/_index.md` is declared `status: active` but contains only 10 lines of content: a header, a filename convention note, and navigation links. No ADRs have been created. This should be `status: stub` until at least one decision record is filed.

**Suggested fix:** Change `status: active` to `status: stub` in `decisions/_index.md`.

---

## Suggestions (worth considering)

### S-1. Missing pages for key data structures and binaries (10 highest-signal gaps)

These terms appear across many pages without wikilinks and without dedicated pages. Ordered by mention frequency:

| Concept | Mentions | Notes |
|---------|----------|-------|
| `DB_VALUE` | 316 | Central union type; mentioned everywhere but has no page |
| `csql` | 194 | Interactive shell binary; `components/csql-shell` exists but bare `csql` is never linked |
| `SERVER_MODE` / `SA_MODE` / `CS_MODE` | 120 / 124 / 76 | Preprocessor guards; `concepts/Build Modes (SERVER SA CS)` covers these but they're never linked by guard name |
| `PT_NODE` | 148 | Parse-tree node struct; `components/parse-tree` exists but `PT_NODE` itself has no anchor page |
| `cub_server` | 79 | Main server binary; no dedicated page |
| `stored procedure` / `Java SP` | 32 / 9 | `components/sp` and `modules/pl_engine` cover this, but the concept name has no page |
| `er_set` | 31 | Core error-set macro; `components/error-manager` exists but `er_set` is unlinked |
| `db_private_alloc` / `parser_alloc` / `free_and_init` | 37 / 28 / 19 | Memory idioms; `concepts/Memory Management Conventions` covers these but they appear unlinked in source pages |
| `PAGE_BUFFER` | 25 | `components/page-buffer` exists but the macro/type name is never linked |
| `replication` | 20 | No HA/replication page exists; mentioned in Data Flow and multiple component pages |

**Suggested fix (highest ROI):** Create a stub at `wiki/concepts/Key Data Structures.md` covering `DB_VALUE`, `PT_NODE`, `PAGE_BUFFER`, `LOCK_RESOURCE`, `LOG_RECORD_HEADER`, `MVCC_TRANS_STATUS` with one-line descriptions and links to owning component pages. This single page would resolve the top unlinked mentions.

---

### S-2. Missing cross-references — `Build Modes` and `Error Handling` pages not linked in component pages

`concepts/Build Modes (SERVER SA CS)` and `concepts/Error Handling Convention` are the canonical references for patterns that appear throughout the component pages. However, most component pages never link to them — they reference `SERVER_MODE`, `SA_MODE`, `er_set` in prose without a wikilink.

**Sample pages lacking the Build Modes link:** `components/query-executor`, `components/scan-manager`, `components/storage`, `components/transaction` (all mention `SERVER_MODE` without linking).

**Suggested fix:** Add a "See also" or "Patterns" section to the 20+ highest-traffic component pages with links to the relevant concept pages.

---

### S-3. Missing cross-references — `Architecture Overview` unlinked in component pages

`Architecture Overview` is referenced by name in 9 component pages without a wikilink. Component pages that describe "how this fits into the overall architecture" should link to it.

Unlinked in: `components/communication`, `components/method`, `components/base`, `components/monitor`, `components/object`.

**Suggested fix:** Add `[[Architecture Overview]]` wikilinks where the plain-text phrase appears.

---

### S-4. `overview.md` is stale and misleading

`wiki/overview.md` still describes the vault as the "claude-obsidian demo vault" with "Sources ingested: 2 / Wiki pages: 26 / Last activity: 2026-04-08". The actual state is 209 pages, Mode B (CUBRID), with 30+ CUBRID source ingests completed. The page was written before the CUBRID scope was adopted.

**Suggested fix:** Rewrite `overview.md` to reflect Mode B scope, current page count (~209), and the CUBRID primary use case. Retire or archive the "demo vault" framing.

---

### S-5. `replication` and HA mode — no page despite 20+ mentions

`replication`, `high availability`, and `HA mode` are mentioned across `Data Flow`, `modules/src`, `components/transaction`, `components/heartbeat`, and `components/log-manager` without a dedicated page or even a stub. This is a significant subsystem with no wiki coverage.

**Suggested fix:** Create `wiki/components/ha-replication.md` as a stub, and add it to `components/_index`.

---

### S-6. `cubrid-src-query-parallel` missing from `sources/_index` and barely linked

`sources/cubrid-src-query-parallel.md` exists and is a complete ingest summary of `src/query/parallel/`. It is not in `sources/_index` and is only referenced by a plain-text path in `log.md` (no wikilink). It is effectively invisible in the graph.

**Suggested fix:** Add to `sources/_index` under CUBRID Codebase Sources. Update `log.md` line 84 to use `[[sources/cubrid-src-query-parallel|cubrid-src-query-parallel]]`.

---

### S-7. Naming convention drift (vault-wide note, not individual fixes)

The vault has two naming conventions in use simultaneously:
- **Hub and concept pages:** Title Case with spaces (`Architecture Overview.md`, `Build Modes (SERVER SA CS).md`, `Error Handling Convention.md`)
- **Component, module, source, dependency pages:** kebab-case (`components/lock-manager.md`, `modules/src.md`, `dependencies/lz4.md`)

This is intentional by design (human-facing hub pages use Title Case; machine-generated content pages use kebab-case) but causes inconsistency in how `_index` pages link to them. The `modules/_index` in particular mixes both styles.

**No individual fixes recommended.** Document the dual convention explicitly in `getting-started.md` or a new `wiki/meta/conventions.md` page so contributors know which style to use when creating new pages.

---

### S-8. `flows/` pages are orphaned from `Data Flow` hub

`flows/dml-execution-path` and `flows/ddl-execution-path` are linked from `flows/_index` (via table escape syntax, correctly), but `Data Flow.md` — the hub page — only links to `[[Query Processing Pipeline]]` and mentions flows generically without linking to the specific flow pages. New readers who arrive at the hub will not discover the detailed flow pages.

**Suggested fix:** Add direct links from `Data Flow.md` to `[[flows/dml-execution-path]]` and `[[flows/ddl-execution-path]]`.

---

### S-9. `meta/claude-obsidian-v1.4-release-session` has empty section headings

`wiki/meta/claude-obsidian-v1.4-release-session.md` has three empty headings: `What Was Built`, `Legal & Security`, and `Visual / README`. These are structural placeholders from the session template.

**Suggested fix:** Fill in the sections or remove the empty headings.

---

### S-10. Dependency pages lack prose content

All 10 `dependencies/` pages (e.g., `dependencies/flex-bison.md`, `dependencies/jansson.md`) have rich YAML frontmatter but empty Markdown body sections. The heading exists but there is no explanatory prose. This makes the pages invisible in reading view and graph traversal beyond their links.

**Suggested fix:** Add 2–3 sentences to each dependency page explaining its role in the CUBRID build and any integration quirks (version mismatches, fork status, etc.). The raw data is already in `sources/cubrid-3rdparty`.

---

## Confirmed Contradictions

### Contradiction 1: `wait_for_graph.c` ownership claim (AGENTS.md vs. reality)

- **Claim (AGENTS.md, surfaced in `sources/cubrid-AGENTS`):** The project guide implies `wait_for_graph.c` is the deadlock detection implementation.
- **Reality (confirmed by `sources/cubrid-src-transaction`, `components/deadlock-detection`, `hot.md`):** `wait_for_graph.c` is compiled only under `#if defined(ENABLE_UNUSED_FUNCTION)` — it is dead code. The active deadlock detector is `lock_deadlock_detect_daemon` embedded in `lock_manager.c`.
- **Status in wiki:** Correctly documented in `components/deadlock-detection` and `sources/cubrid-src-transaction`. Still appears uncorrected in `components/transaction.md` lines 74 and 95 (see C-6 above).

### Contradiction 2: `strict_warnings` listed in AGENTS.md but absent from tree

- **Claim (AGENTS.md):** `src/debugging/` owns `strict_warnings` as a file.
- **Reality (confirmed by `sources/cubrid-src-debugging`):** No file named `strict_warnings.*` exists in the tree under any extension.
- **Status in wiki:** Correctly flagged in `components/debugging` and `sources/cubrid-src-debugging`. Partially ambiguous in `components/_index.md` line 216 (see C-7 above).

---

## Index.md Stale Entry Check

| Entry | Status |
|-------|--------|
| `[[Wiki Map]]` | DEAD — page does not exist; referenced twice in `index.md` frontmatter and twice in body |
| All other 48 index entries | OK |

---

## Vault Health Assessment

The vault is **functional but carries a meaningful backlog.** No data is lost and the core CUBRID knowledge graph is intact and well-connected. The issues cluster into two types:

**Systematic gaps (not ingest quality problems):** The 14 driver/submodule module stubs (`cubrid-cci`, `cubrid-jdbc`, `cubridmanager`, `cs`, `sa`, `cubrid`, `conf`, `win`, `cm_common`) and the 56 frontmatter gaps are structural holes from an ingest run that covered `src/` thoroughly but did not yet reach the client-driver or build-support directories. These will resolve naturally as ingests continue.

**Editorial gaps (require deliberate action):** The two confirmed contradictions (C-6 `wait_for_graph.c`, C-7 `strict_warnings`) need a human to write the correcting sentences. The `overview.md` stale content (S-4) will actively mislead new readers. The `[[Wiki Map]]` dead link on the front page (C-5) is a visible broken experience on every vault open.

**Immediate priority actions (top 5):**
1. Fix `modules/src.md` underscore typos (C-4) — two-line edit
2. Remove or replace `[[Wiki Map]]` from `index.md`, `getting-started.md`, `concepts/_index.md` (C-5)
3. Correct the `wait_for_graph.c` misleading diagram and table entry in `components/transaction.md` (C-6)
4. Rewrite `overview.md` to reflect Mode B reality (S-4)
5. Add `sources/cubrid-src-query-parallel` and the 8 other missing sources to `sources/_index.md` (W-8)
