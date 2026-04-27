---
type: meta
title: "Wiki Index"
created: 2026-04-07
updated: 2026-04-24
tags:
  - meta
  - index
status: evergreen
related:
  - "[[overview]]"
  - "[[log]]"
  - "[[hot]]"
  - "[[dashboard]]"
  - "[[Wiki Map.canvas]]"
  - "[[Architecture Overview]]"
  - "[[Tech Stack]]"
  - "[[Data Flow]]"
  - "[[Dependency Graph]]"
  - "[[Key Decisions]]"
  - "[[modules/_index]]"
  - "[[components/_index]]"
  - "[[decisions/_index]]"
  - "[[dependencies/_index]]"
  - "[[flows/_index]]"
  - "[[prs/_index]]"
  - "[[_legacy/_index]]"
---

# Wiki Index

Last updated: 2026-04-24 | Mode: B (CUBRID codebase)

Navigation: [[overview]] | [[log]] | [[hot]] | [[dashboard]] | [[Wiki Map.canvas|Wiki Map]]

---

## CUBRID (Mode B)

Hub pages:
- [[Architecture Overview]] — topology, subsystems, entry point
- [[Tech Stack]] — languages, build, bundled libs
- [[Data Flow]] — query path and lifecycles
- [[Dependency Graph]] — internal + external deps
- [[Key Decisions]] — ADR roll-up

Sub-indexes:
- [[modules/_index|Modules]] — per top-level source directory
- [[components/_index|Components]] — subsystems (optimizer, lock mgr, …)
- [[decisions/_index|Decisions]] — ADRs
- [[dependencies/_index|Dependencies]] — external / bundled libs
- [[flows/_index|Flows]] — request paths
- [[prs/_index|PRs]] — documented merged upstream PRs

---

## Concepts

- [[Query Processing Pipeline]] — SQL → lexer → parser → name resolution → semantic check → XASL → execute
- [[Build Modes (SERVER SA CS)]] — same source, three binaries via preprocessor guards
- [[Memory Management Conventions]] — `free_and_init`, `db_private_alloc`, `parser_alloc`; no RAII
- [[Error Handling Convention]] — C-style codes, six-place new-error-code rule
- [[Code Style Conventions]] — CI-enforced formatting & naming

---

## Entities

- [[CUBRID]] — open-source C/C++17 RDBMS with Java PL engine; Apache 2.0; v11.4.0.1778

---

## CUBRID 11.4 User Manual (English)

End-user reference companion to the source-code wiki. **Hub**: [[sources/cubrid-manual-en-overview]]. 22 source pages catalog the full manual (~88K lines).

- **General**: [[sources/cubrid-manual-intro]] · [[sources/cubrid-manual-install]] · [[sources/cubrid-manual-csql]]
- **HA / Sharding / Security**: [[sources/cubrid-manual-ha]] · [[sources/cubrid-manual-shard]] · [[sources/cubrid-manual-security]]
- **Admin**: [[sources/cubrid-manual-admin]] · [[sources/cubrid-manual-config-params]] · [[sources/cubrid-manual-error-codes]]
- **API drivers**: [[sources/cubrid-manual-api]] · [[sources/cubrid-manual-cci]] · [[sources/cubrid-manual-jdbc]]
- **Stored procedures**: [[sources/cubrid-manual-pl]] · [[sources/cubrid-manual-plcsql]]
- **SQL reference**: [[sources/cubrid-manual-sql-foundation]] · [[sources/cubrid-manual-sql-dml]] · [[sources/cubrid-manual-sql-ddl]] · [[sources/cubrid-manual-sql-tuning-parallel]] · [[sources/cubrid-manual-sql-functions]]
- **Release notes**: [[sources/cubrid-manual-release-notes-114]]

---

## Legacy seed (pre-CUBRID)

Pages about the LLM Wiki pattern itself and the claude-obsidian plugin's release history have been moved to `wiki/_legacy/`. See [[_legacy/_index|Legacy Seed Index]].
