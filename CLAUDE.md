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

### PR Ingest (user-specified only, all states accepted, code analysis required)

PR ingest is **strictly on-demand**. Only run this flow when the user explicitly names a PR (e.g. "ingest PR #NNNN", "PR #NNNN 분석해줘"). **Never** autonomously scan `gh pr list`, poll for recent merges, batch-ingest, or run a background loop. If you notice an unreferenced PR, do not ingest — ask or stay silent. The user may later wire a `/loop` or `/schedule`; until then, one PR per explicit request.

Every PR state is ingestable — but behavior differs. Merged PRs can update the wiki and bump baseline; open/draft/approved PRs are documented with a **Reconciliation Plan only** (no component-page edits for PR-induced changes); closed-without-merge PRs are documented as abandoned. In all cases, **deep code analysis of the touched files is required** and **Incidental Knowledge Enhancement** of the wiki is expected (see step 5).

Inferred repo: `git -C ~/dev/cubrid/ config --get remote.origin.url` (currently `https://github.com/CUBRID/cubrid`). Reinfer every ingest; do not hard-code.

**Protocol:**

1. **Fetch PR data.** All via `gh` CLI:
   ```
   gh pr view NNNN --repo CUBRID/cubrid --json number,title,body,author,state,url,baseRefName,headRefName,baseRefOid,headRefOid,mergeCommit,mergedAt,closedAt,isDraft,files,commits,reviews,comments > /tmp/pr-NNNN.json
   gh pr diff NNNN --repo CUBRID/cubrid > /tmp/pr-NNNN.diff
   gh api repos/CUBRID/cubrid/pulls/NNNN/comments --paginate > /tmp/pr-NNNN.review-comments.json   # inline review threads (line-level)
   ```

2. **Classify PR state and (if merged) relation to baseline.** Parse `state` + `isDraft` from the JSON:
   - `MERGED` → resolve `mergeCommit.oid`, then classify vs current baseline:
     - Case **(a)** `merge_commit == baseline` → retroactive doc, no PR-reconciliation, no bump.
     - Case **(b)** `git -C ~/dev/cubrid/ merge-base --is-ancestor <merge_commit> <baseline>` exits 0 → already absorbed; retroactive doc, no PR-reconciliation, no bump.
     - Case **(c)** `git -C ~/dev/cubrid/ merge-base --is-ancestor <baseline> <merge_commit>` exits 0 → newer than baseline; full PR-reconciliation + baseline bump.
     - Case **(d)** neither ancestor relationship → divergent / force-push / rebase. **Stop** and ask the user.
   - `OPEN` (non-draft) → `status: open`. Write Reconciliation Plan; do NOT apply it.
   - `OPEN` + `isDraft: true` → `status: draft`. Same as open but tagged; plan may be rougher.
   - `CLOSED` + not merged → `status: closed-unmerged`. Document as abandoned; no reconciliation plan (nothing to apply — change will never land).

   Note: "PR-reconciliation" ≠ "Incidental Knowledge Enhancement". See step 5.

3. **Deep code analysis.** Not just `gh pr diff`. For every file in `files_changed`:
   a. Read the **baseline** file at `~/dev/cubrid/<path>` (current on-disk state reflects the current baseline for this vault). For merged-case-(c) ingests, also read the **post-merge** state by spot-checking `git -C ~/dev/cubrid/ show <merge_commit>:<path>` where the diff is non-trivial.
   b. Read surrounding context: callers of the modified functions, headers of touched types, related files in the same directory. Use `grep -rn '<symbol>' ~/dev/cubrid/src/` liberally.
   c. For every meaningful edit in the diff, understand **why** — what invariant does this preserve, what regression does this guard, what API contract changes.
   d. Record structural observations: new/removed/renamed functions/types/macros; changed signatures; changed defaults; new invariants or ordering requirements; performance characteristics (big-O, contention behavior, locking).
   e. Extract `[!contradiction]` signals: code that conflicts with an existing wiki claim → note for follow-up.

4. **Create the PR page.** `wiki/prs/PR-NNNN-<slug>.md` from `_templates/pr.md`. Slug: kebab-case, ≤6 words, derived from PR title. Fill frontmatter per template; set `status` to the PR state value from step 2; set `ingest_case` to `a`/`b`/`c`/`d`/`open`/`draft`/`closed-unmerged`. Populate body sections:
   - Summary, Motivation, Changes (Structural / Behavioral / Per-file notes) — synthesize, don't paste raw diff/body
   - Review discussion highlights — only authoritative rationale; skip nits and bot comments
   - **Reconciliation Plan** (REQUIRED for `open`/`draft`; "n/a" for merged-cases-(a)/(b) and closed-unmerged; "Applied during this ingest — see below" for merged-case-(c))
   - **Pages Reconciled** (merged-case-(c) only)
   - **Incidental wiki enhancements** (always — may be "none")
   - Baseline impact, Related

5. **Incidental Knowledge Enhancement (applies to every PR state).** While doing step 3's analysis, whenever you find a fact about the **baseline** code that is missing, incorrect, or incomplete in the existing wiki — **apply the enhancement now**, regardless of PR state.
   - Target page selection: the most specific owning page first (component/*), then source/*, then hub pages if truly cross-cutting.
   - Edit discipline: add a prose paragraph or bullet; do not restructure the page. Cite briefly ("noted while analyzing PR #NNNN") only when the PR is the obvious prompt; otherwise present as baseline truth without PR attribution.
   - Use `[!contradiction]` when code conflicts with a wiki claim; use `[!gap]` when the wiki had a known hole now being filled.
   - List every page edited under the PR page's "Incidental wiki enhancements" section with a one-line summary per page.
   - **This step is orthogonal to PR-reconciliation.** An open PR contributes no PR-reconciliation edits, but its analysis can still enrich the wiki with baseline-truths the author's reading happened to surface.

6. **PR-Reconciliation (merged-case-(c) only).** For each file in `files_changed`:
   a. `grep -rn '<filename-basename>' wiki/ --include='*.md'` + `grep -rn '<key-symbol>' wiki/`.
   b. For each hit, determine whether the existing claim is still accurate post-PR.
   c. Edit stale sections, adding a `> [!update]` callout citing `PR #NNNN` and the merge-commit short-sha. Removed/renamed items get `> [!contradiction]` with before/after commit hashes.
   d. Files with zero wiki references → flag in the PR page's "Changes / new surface" subsection (candidate for future dedicated ingest).

7. **Reconciliation Plan generation (open/draft only).** Do NOT edit component pages. Instead, inside the PR page body write a "## Reconciliation Plan" section containing, per affected wiki page:
   - Page link (wikilink) + section anchor that would change
   - Concrete before/after excerpts (quote the existing claim, show the proposed replacement)
   - Rationale (which diff hunk or review thread justifies the edit)
   - Suggested callout type (`[!update]` / `[!contradiction]` / `[!gap]`)

   Also list "new surface — no existing wiki reference" entries here.

   The plan must be executable later without re-reading the PR — it is the single source of truth for a deferred reconciliation.

8. **Baseline bump (merged-case-(c) only).**
   a. Rewrite the full hash in this file (`CLAUDE.md` § "CUBRID Baseline Commit").
   b. Rewrite the full hash in `wiki/hot.md` § "CUBRID Baseline Commit".
   c. Prepend a log entry to `wiki/log.md`: `## [YYYY-MM-DD] baseline-bump | <old-sha7> → <new-sha7> (PR #NNNN)` with a bullet list of reconciled pages.
   d. Set `triggered_baseline_bump: true`, `baseline_before: <old>`, `baseline_after: <new>` in the PR page frontmatter.

9. **Deferred-plan execution (separate invocation).** When the user later says "apply reconciliation for PR #NNNN" (or "execute plan for PR #NNNN"):
   - Re-fetch PR state. If now merged, detect case vs current baseline and run the merged flow from step 2 (the existing plan becomes the starting point, but revalidate — the PR may have changed before merge).
   - If still open/draft, execute the plan against the current baseline anyway (the user is opting in to premature reconciliation). Flag in the commit message that this was a pre-merge reconciliation.
   - In both cases, after execution set `reconciliation_applied: true`, `reconciliation_applied_at: YYYY-MM-DD` in the PR page frontmatter; promote the "Reconciliation Plan" section content to "Pages Reconciled" (keep it in the page for audit).

10. **Optional companion ADR.** If the PR represents a major design choice (API break, new subsystem, semantic shift, a design convergence worth preserving), file `wiki/decisions/NNNN-<slug>.md` citing the PR page. Judgment call.

`gh` auth: confirm via `gh auth status`; if unauthenticated, prompt the user to run `!gh auth login`. Do not authenticate on their behalf.

**Summary of when each output happens**

| PR state | PR page | PR-reconciliation | Reconciliation Plan | Incidental enhancements | Baseline bump |
|---|---|---|---|---|---|
| merged, case (a) equal | yes | no | n/a | yes | no |
| merged, case (b) absorbed | yes | no | n/a | yes | no |
| merged, case (c) newer | yes | **yes, now** | promoted to "Pages Reconciled" | yes | **yes** |
| merged, case (d) divergent | stop, ask | — | — | — | — |
| open / approved | yes | no | **yes, written** | yes | no |
| draft | yes | no | yes, written (may be rougher) | yes | no |
| closed-unmerged | yes | no | no (nothing to apply) | yes (if any surfaced) | no |

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
