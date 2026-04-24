# claude-obsidian — Claude + Obsidian Wiki Vault

This folder is both a Claude Code plugin and an Obsidian vault.

**Plugin name:** `claude-obsidian`
**Skills:** `/wiki`, `/wiki-ingest`, `/wiki-query`, `/wiki-lint`
**Vault path:** This directory (open in Obsidian directly)

## What This Vault Is For

Primary scope: **Documenting the CUBRID relational database source tree** at `~/dev/cubrid/` (Mode B — GitHub / codebase wiki). Captures modules, components, data flows, decisions, and dependencies.

Secondary scope: A small seed cluster of pages about the LLM Wiki pattern itself (how this vault works) has been archived under `wiki/_legacy/` (see [[_legacy/_index]]). These predate the CUBRID scope and are retained as meta-documentation; do not extend them — all new content goes into the Mode B CUBRID structure below.

## Mode B Conventions (CUBRID)

Structure under `wiki/`:
- `modules/` — one page per top-level CUBRID directory (broker, src, cs, pl_engine, ...)
- `components/` — subsystems (optimizer, page buffer, lock manager, ...)
- `decisions/` — ADRs (`NNNN-short-title.md`)
- `dependencies/` — one page per external / bundled library
- `flows/` — request paths, lifecycles, recovery sequences
- `prs/` — documented merged upstream PRs (`PR-NNNN-short-slug.md`); see PR Ingest protocol below

Hub pages at `wiki/` root:
- [[Architecture Overview]] · [[Tech Stack]] · [[Data Flow]] · [[Dependency Graph]] · [[Key Decisions]]

Source of truth for CUBRID: `~/dev/cubrid/` — **never write to the source tree**, only read.

### CUBRID Baseline Commit

**All wiki content under `wiki/` (outside `_legacy/`) is anchored to CUBRID commit `175442fc858bd0075165729756745be6f8928036`.** Every claim, file path, line number, and structural observation reflects the source tree at that commit.

**Before any new CUBRID ingest, analysis, or wiki update, you MUST:**

1. Check the current HEAD of the CUBRID source tree:
   ```
   git -C ~/dev/cubrid/ rev-parse HEAD
   ```
2. If HEAD == `175442fc858bd0075165729756745be6f8928036`, proceed normally.
3. If HEAD is **newer** (i.e. `git -C ~/dev/cubrid/ merge-base --is-ancestor 175442fc858bd0075165729756745be6f8928036 HEAD` exits 0), do this before writing anything:
   a. Compute the delta for the path you are about to ingest/update:
      ```
      git -C ~/dev/cubrid/ log --oneline 175442fc858bd0075165729756745be6f8928036..HEAD -- <path>
      git -C ~/dev/cubrid/ diff --stat 175442fc858bd0075165729756745be6f8928036..HEAD -- <path>
      ```
   b. For each changed file in that delta, grep existing wiki pages for references to the file path or affected symbols (`grep -rn '<filename>\|<symbol>' wiki/ --include='*.md'`).
   c. Update those wiki pages to reflect the new state. Flag removed/renamed items with `> [!contradiction]` or `> [!gap]` callouts citing both commits.
   d. After all affected pages are reconciled, update the baseline hash in this file (`CLAUDE.md`) and in `wiki/hot.md` to the new HEAD, and log the bump in `wiki/log.md` under `## [YYYY-MM-DD] baseline-bump | <old-sha7> → <new-sha7>` with the list of files reconciled.
4. If HEAD is **older** or on a divergent branch, stop and ask the user — do not silently proceed.

This rule supersedes "just ingest" behavior: the baseline is authoritative, drift must be reconciled before new content lands, and the baseline only moves forward after reconciliation.

### PR Ingest (merged PRs only, user-specified only)

PR ingest is **strictly on-demand**. Only run this flow when the user explicitly names a PR (e.g. "ingest PR #NNNN", "파일 PR #NNNN"). **Never** autonomously scan `gh pr list`, poll for recent merges, batch-ingest, or run a background loop over PRs on your own initiative. If you notice an unreferenced merged PR, do not ingest it — ask the user first or stay silent. If the user later sets up a cron/loop themselves via `/loop` or `/schedule`, that is their decision; until then, one PR per explicit request.

When the user says "ingest PR #NNNN" (or equivalent), it refers to an upstream `CUBRID/cubrid` pull request. **Only merged PRs are accepted** — open / draft / closed-without-merge PRs are proposed or abandoned changes, not part of the "what the code is" record, and must not land in the wiki.

Inferred repo: `git -C ~/dev/cubrid/ config --get remote.origin.url` (currently `https://github.com/CUBRID/cubrid`). Do not hard-code; reinfer on every ingest.

**Protocol:**

1. Fetch PR metadata and diff (all via `gh`; no web scraping, no MCP needed):
   ```
   gh pr view NNNN --repo CUBRID/cubrid --json number,title,body,author,state,url,baseRefName,headRefName,baseRefOid,headRefOid,mergeCommit,mergedAt,files,commits,reviews,comments > /tmp/pr-NNNN.json
   gh pr diff NNNN --repo CUBRID/cubrid > /tmp/pr-NNNN.diff
   ```
2. **Gate on merged state.** Parse `state` from the JSON. If `state != "MERGED"`, **stop** and report the state to the user. Do not create any wiki page.
3. **Classify the PR's merge commit relative to the current baseline** (baseline = the hash recorded in this file above):
   - `merge_commit = mergeCommit.oid`
   - Case (a) `merge_commit == baseline` — wiki already at this state; ingest as documentation only, no reconciliation, no bump.
   - Case (b) `git -C ~/dev/cubrid/ merge-base --is-ancestor <merge_commit> <baseline>` exits 0 — the PR was already absorbed into an earlier bump. Ingest as retroactive documentation: create the `wiki/prs/PR-NNNN-<slug>.md` page but skip reconciliation and skip bump. Note `triggered_baseline_bump: false` in frontmatter.
   - Case (c) `git -C ~/dev/cubrid/ merge-base --is-ancestor <baseline> <merge_commit>` exits 0 — the PR is newer than baseline. Full reconciliation + bump (see step 5).
   - Case (d) neither ancestor relationship holds — divergent branch / force-push / rebase. **Stop** and ask the user how to proceed.
4. **Create `wiki/prs/PR-NNNN-<slug>.md`** using `_templates/pr.md` as the skeleton. Slug: kebab-case, derived from PR title, max ~6 words. Fill frontmatter from the JSON: `pr_number`, `pr_url`, `repo`, `author`, `merged_at`, `merge_commit`, `base_ref`, `head_ref`, `base_sha`, `head_sha`, `files_changed`. Populate body sections (Summary, Motivation, Changes, Review discussion highlights) by synthesizing — never paste the raw PR body or raw diff verbatim.
5. **Reconciliation (Case c only).** For each file in `files_changed`:
   a. `grep -rn '<filename-basename>' wiki/ --include='*.md'` — find every wiki page that references it.
   b. Read each hit's context. Determine whether the claim is still accurate post-PR.
   c. For pages that need updating, edit the relevant section and add a `> [!update]` callout citing `PR #NNNN` and the merge commit short-sha. For removed/renamed items, use `> [!contradiction]` with before/after commit hashes.
   d. If a changed file has **no** existing wiki reference, note it in the PR page's "Wiki impact" section as "new surface — not yet documented" (candidate for future ingest).
6. **Baseline bump (Case c only).** After all affected pages are reconciled:
   a. Update the baseline hash in this file (`CLAUDE.md` § "CUBRID Baseline Commit") — replace the old full hash.
   b. Update the baseline hash in `wiki/hot.md` § "CUBRID Baseline Commit".
   c. Append an entry to `wiki/log.md` at the TOP under `## [YYYY-MM-DD] baseline-bump | <old-sha7> → <new-sha7> (PR #NNNN)` with a bulleted list of the reconciled pages.
   d. Set `triggered_baseline_bump: true`, `baseline_before: <old>`, `baseline_after: <new>` in the PR page frontmatter.
7. **Optional companion ADR.** If the PR represents a major design choice (new subsystem, API break, behavioral semantics shift), file an accompanying `wiki/decisions/NNNN-<slug>.md` citing the PR page. Judgment call — not every PR earns an ADR.

`gh` auth: the user runs `gh auth status` to confirm; if unauthenticated, prompt them to run `gh auth login` in the terminal (`!gh auth login` in chat). Do not attempt to authenticate on their behalf.

## Vault Structure

```
.raw/               ingestable documents (PDFs, transcripts, articles) — immutable; Claude reads but never modifies
wiki/               Claude-generated knowledge base (CUBRID Mode B)
wiki/prs/           documented merged CUBRID upstream PRs (PR-NNNN-<slug>.md)
wiki/_legacy/       archived pre-CUBRID seed (LLM Wiki pattern meta-docs; do not extend)
_templates/         Obsidian Templater templates (component.md, decision.md, pr.md, ...)
_attachments/       images and PDFs referenced by wiki pages
```

The CUBRID source tree lives **outside this vault** at `~/dev/cubrid/` and is read directly by absolute path. **Do not create a `.raw/cubrid/` symlink** — `.raw/` is for documents to ingest, not for source-tree access. Older wiki pages may still mention such a symlink as a Mac-era convention; treat those as historical and reference `~/dev/cubrid/<path>` directly going forward.

## How to Use

Drop a source file into `.raw/`, then tell Claude: "ingest [filename]".

Ask any question. Claude reads the index first, then drills into relevant pages.

Run `/wiki` to scaffold a new vault or check setup status.

Run "lint the wiki" every 10-15 ingests to catch orphans and gaps.

## Cross-Project Access

To reference this wiki from another Claude Code project, add to that project's CLAUDE.md:

```markdown
## Wiki Knowledge Base
Path: /path/to/this/vault

When you need context not already in this project:
1. Read wiki/hot.md first (recent context, ~500 words)
2. If not enough, read wiki/index.md
3. If you need domain specifics, read wiki/<domain>/_index.md
4. Only then read individual wiki pages

Do NOT read the wiki for general coding questions or things already in this project.
```

## Plugin Skills

| Skill | Trigger |
|-------|---------|
| `/wiki` | Setup, scaffold, route to sub-skills |
| `ingest [source]` | Single or batch source ingestion |
| `query: [question]` | Answer from wiki content |
| `lint the wiki` | Health check |
| `/save` | File the current conversation as a structured wiki note |
| `/autoresearch [topic]` | Autonomous research loop: search, fetch, synthesize, file |
| `/canvas` | Visual layer: add images, PDFs, notes to Obsidian canvas |

## MCP (Optional)

If you configured the MCP server, Claude can read and write vault notes directly.
See `skills/wiki/references/mcp-setup.md` for setup instructions.
