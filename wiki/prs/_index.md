---
type: index
title: "CUBRID PRs (Merged)"
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

# CUBRID Merged Pull Requests

Navigation: [[index]] | [[modules/_index|Modules]] | [[components/_index|Components]] | [[decisions/_index|Decisions]]

Documented merged PRs from the CUBRID upstream (`CUBRID/cubrid`). Each PR page captures:
- Motivation + summary of the change
- Files + structural + behavioral impact
- Authoritative review discussion (design rationale only)
- Which wiki pages were reconciled because of it
- Baseline-bump before/after hashes

**Only merged PRs are ingested, and only when the user explicitly names one.** Claude does not scan, poll, or batch PRs on its own initiative. Open PRs are upcoming-change proposals, not part of the "what the code is" record. See `CLAUDE.md` § "PR Ingest (merged PRs only, user-specified only)" for the full protocol.

Filename convention: `PR-NNNN-short-slug.md` where `NNNN` is the upstream PR number.

## Merged PRs

<!-- Newest first. Added automatically by the ingest protocol. -->

_(none yet — first ingest will populate this list)_

---

## Relationship to other pages

- **[[decisions/_index|Decisions]]** — when a PR represents a major design choice, a companion ADR is filed under `decisions/` citing the PR page.
- **[[components/*]]** — PR ingest updates component pages with `> [!update]` callouts that cite the PR number and merge commit.
- **[[sources/*]]** — file-level source pages get the same `> [!update]` treatment.
- **[[log]]** — every PR ingest that bumps the baseline produces a `baseline-bump` log entry.
