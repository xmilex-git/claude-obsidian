---
type: meta
title: "Lint Report 2026-04-29 (CBRD-26722 ingest)"
created: 2026-04-29
updated: 2026-04-29
tags:
  - meta
  - lint
status: developing
---

# Lint Report: 2026-04-29 (post CBRD-26722 ingest)

Focused lint following the CBRD-26722 knowledge-dump ingest. Full vault scan — but issues called out below were the ones surfaced by the new content; the rest of the vault inherits the previous lint posture (last full sweep `lint-report-2026-04-24`, last partial-pass `lint-report-2026-04-29-pr7062-reingest` if present).

## Summary
- Pages scanned: 315 markdown files under `wiki/`
- New pages this ingest: 2 ([[components/parallel-index-scan]], [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables]])
- Updated pages this ingest: 5 ([[prs/PR-7062-parallel-scan-all-types]], [[components/_index]], [[sources/_index]], [[log]], [[hot]])
- Issues found: 0 errors, 0 warnings, 1 informational
- Auto-fixed: 0
- Needs review: 0

## Orphan Pages
None. Both new pages have multiple inbound wikilinks:
- [[components/parallel-index-scan]] ← [[components/_index]], [[hot]], [[log]], [[prs/PR-7062-parallel-scan-all-types]] (×2), [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables]] (×3)
- [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables]] ← [[sources/_index]], [[hot]], [[log]], [[prs/PR-7062-parallel-scan-all-types]] (×2), [[components/parallel-index-scan]] (×3)

## Dead Links
None in the new content. Every wikilink in both new pages resolves to an existing page.

## Frontmatter Gaps
None. Both new pages carry the required fields:
- `type`, `status`, `created`, `updated`, `tags`, `address`, `related`
- [[components/parallel-index-scan]] additionally carries `parent_module`, `path`, `purpose`, `key_files`, `public_api` (component schema)
- [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables]] additionally carries `source_path`, `source_branch`, `source_head`, `source_hash`, `source_kind`, `ingested`, `related_pr`, `jira` (source schema)

## Stale Claims
None new. The supersession callout on [[prs/PR-7062-parallel-scan-all-types]] (2026-04-29 prior round) flagging `0f8a107bb`-superseded claims remains valid; the new content explicitly anchors to HEAD `7fdb82099` and adds the four-commit follow-on summary.

## Cross-Reference Gaps
None. Mentions of `parallel-heap-scan`, `parallel-list-scan`, `scan-manager`, `btree`, `xasl`, `partition-pruning`, `query-executor` in the new pages are all wikilinked to their canonical component pages.

## Address Validation

- Counter state: `8` (next allocation will be `c-000008`).
- Highest c- address observed: `c-000007` ([[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables]]).
- Post-rollout pages with addresses: 7 (`c-000001` … `c-000007`); none failing format check; none colliding.
- `.raw/.manifest.json::address_map` consistent — all 5 mapped paths exist on disk and frontmatter `address` matches the mapping.

### Errors
None.

### Pending backfill (informational)
~310 legacy pages (`created:` < 2026-04-23) without addresses. This is the expected DragonScale Phase-2 carry-over; backfill is a separate deliberate operation not part of this ingest.

## Semantic Tiling
Skipped — `scripts/tiling-check.py` is not present in this vault. No change to that posture.

## Notes for the next lint sweep

- Branch-WIP pages now total 3: [[components/parallel-list-scan]], [[flows/parallel-list-scan-open]], [[components/parallel-index-scan]]. All three are tagged `status: branch-wip` and explicitly anchor to branch HEAD `7fdb82099` (or earlier on the same branch). They are intentionally excluded from baseline-truth lint; on PR #7062 merge they will be reconciled into canonical `parallel-scan-input-handler-{list,index}` / `parallel-scan-slot-iterator-{list,index}` pages per the [[prs/PR-7062-parallel-scan-all-types|PR #7062]] Reconciliation Plan.
- The "candidate enhancements not applied" list on [[prs/PR-7062-parallel-scan-all-types]] (MRO/ISS/ILS, no-page-latch→mutex-ordering convention, thread_local-static class-member pattern, deferred-promotion `parallel_pending` pattern) remains carried forward — none added this round; all four would land more naturally as part of the post-merge canonical page set.
- `.vault-meta/legacy-pages.txt` is still missing — record the rollout baseline at the top of that file when convenient (`# rollout: 2026-04-23`).
