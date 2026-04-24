---
type: pr
pr_number:
pr_url:
repo:
state:                          # MERGED | OPEN | CLOSED — matches gh JSON state
is_draft: false
author:
created_at:
merged_at:
closed_at:
merge_commit:
base_ref:
head_ref:
base_sha:
head_sha:
jira:
files_changed: []
related_components: []
related_sources: []
ingest_case:                    # a | b | c | d | open | draft | closed-unmerged
triggered_baseline_bump: false
baseline_before:
baseline_after:
reconciliation_applied: false   # set to true after Reconciliation Plan is executed (for open PRs that later get applied)
reconciliation_applied_at:
incidental_enhancements_count: 0
tags:
  - pr
  - cubrid
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
status:                          # mirror state: merged | open | draft | closed-unmerged
---

# <% tp.file.title %>

> [!info] PR metadata
> **Repo:** `<repo>` · **State:** `<state>` · **Author:** `@<author>` · **Merge commit:** `<merge_commit>` (if merged)
> **Base → Head:** `<base_ref>` (`<base_sha>`) → `<head_ref>` (`<head_sha>`)

<!-- For merged PRs, add a case note: -->
<!-- > [!note] Ingest classification: case (X) -->
<!-- > Explain the relation to baseline (ancestor/descendant/divergent/equal). -->

<!-- For open/draft PRs, add: -->
<!-- > [!note] Ingest classification: open (or draft) -->
<!-- > Reconciliation Plan written below. No component pages were edited for PR-induced changes. Incidental wiki enhancements from baseline analysis WERE applied. -->

## Summary

<!-- One paragraph: what the PR changes and why. Synthesize from title + body; do NOT paste raw body. -->

## Motivation

<!-- The problem being solved. Jira ticket, upstream bug, performance target, review catalyst, etc. -->

## Changes

### Structural

<!-- New/deleted/renamed files. New types/macros/APIs. Signature changes. ABI impact if any. -->

-

### Per-file notes

<!-- One sub-bullet per meaningful file, linking to the owning wiki page. -->

- `src/...` — what changed and why ([[components/X]])
- `src/...` — ([[components/Y]])

### Behavioral

<!-- Semantics: error codes, concurrency, ordering, performance, edge cases. Cite diff lines or file paths. -->

-

### New surface (no existing wiki reference)

<!-- Files/symbols touched by the PR that have no wiki page. Flagged for future dedicated ingest. -->

-

## Review discussion highlights

<!-- Only authoritative design rationale. Skip nits, CI bot output, and "/run all" commands. -->

-

## Reconciliation Plan

<!--
For open/draft PRs: REQUIRED. List every wiki page that would need an update post-merge,
with concrete before/after excerpts so the plan is executable later without re-reading the PR.

Structure per affected page:

### [[components/X]] — section anchor name
- **Current claim:** "<quote from existing wiki page>"
- **Proposed replacement:** "<new prose or diff>"
- **Rationale:** which PR hunk or review thread justifies this
- **Callout type:** [!update] | [!contradiction] | [!gap]

For merged-case-(a)/(b): "n/a — PR's changes already reflected in current baseline."
For merged-case-(c): "Applied during this ingest — see Pages Reconciled below."
For closed-unmerged: "n/a — change will never land."
-->

## Pages Reconciled

<!--
For merged-case-(c) only. List each component/source page edited with a one-line summary
of what was changed and which callout type was added. Leave blank / "none" otherwise.

For open PRs whose Reconciliation Plan was later executed, promote the plan content here and
note reconciliation_applied_at.
-->

-

## Incidental wiki enhancements

<!--
REQUIRED regardless of PR state. Facts about the BASELINE code that were missing, incorrect, or
incomplete in the wiki and that were added/corrected during the deep-analysis phase of this ingest.

Structure per enhancement:

- [[components/Y]] — added note: "<one-line summary of what was added>"
- [[sources/cubrid-src-Z]] — corrected claim: "<one-line summary>"
- [[components/W]] — [!contradiction] filed: "<one-line summary>"

If none surfaced, state "none" explicitly (so the audit trail is explicit, not ambiguous).
-->

-

## Baseline impact

- Before: `<baseline_before>`
- After: `<baseline_after>`  (same as before if no bump)
- Bump triggered: `<true|false>`
- Logged: [[log]] under `[YYYY-MM-DD] baseline-bump` (if applicable)

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: `<pr_url>`
- Jira: `<jira>`
- Components: 
- Sources: 
