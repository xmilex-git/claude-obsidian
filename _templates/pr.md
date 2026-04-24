---
type: pr
pr_number:
pr_url:
repo:
state: merged
author:
merged_at:
merge_commit:
base_ref:
head_ref:
base_sha:
head_sha:
files_changed: []
related_components: []
related_sources: []
triggered_baseline_bump: false
baseline_before:
baseline_after:
tags:
  - pr
  - cubrid
created: <% tp.date.now("YYYY-MM-DD") %>
updated: <% tp.date.now("YYYY-MM-DD") %>
status: merged
---

# <% tp.file.title %>

> [!info] PR metadata
> **Repo:** `<repo>` · **Merged:** `<merged_at>` by `<author>` · **Merge commit:** `<merge_commit>`
> **Base → Head:** `<base_ref>` (`<base_sha>`) → `<head_ref>` (`<head_sha>`)

## Summary

<!-- One paragraph. What the PR changes and why. Synthesized from PR title + description. Do not paste the raw PR body verbatim. -->

## Motivation

<!-- The problem the PR solves. From description, linked issues, or commit messages. -->

## Changes

### Files changed

<!-- Grouped by subsystem. Link each to the component/source page it affects. -->

- `path/to/file1.c` — what changed at the symbol/function level ([[components/X]])
- `path/to/file2.h` — what changed ([[components/Y]])

### Structural changes

<!-- New files, renames, deletes, API changes, enum additions, ABI impact. -->

-

### Behavioral changes

<!-- Semantics: error codes, concurrency, performance, edge cases. Cite diff lines. -->

-

## Review discussion highlights

<!-- Only authoritative design-rationale comments. Skip nits. Cite reviewer handles. -->

-

## Wiki impact

### Pages updated in this reconciliation

<!-- List every wiki page touched as part of the baseline bump triggered by this PR. -->

- [[components/...]] — what claim changed
- [[sources/cubrid-src-...]] — what file-level observation changed

### Baseline bump

- Before: `<baseline_before>`
- After: `<baseline_after>` (this PR's merge commit)
- Logged in: [[log]] under `[YYYY-MM-DD] baseline-bump`

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: `<pr_url>`
- Components: 
- Sources: 
