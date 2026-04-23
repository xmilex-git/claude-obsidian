---
type: meta
title: "Wiki Index"
updated: 2026-04-23
tags:
  - meta
  - index
status: evergreen
related:
  - "[[overview]]"
  - "[[log]]"
  - "[[hot]]"
  - "[[dashboard]]"
  - "[[Wiki Map]]"
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
---

# Wiki Index

Last updated: 2026-04-23 | Mode: B (CUBRID codebase) + legacy seed

Navigation: [[overview]] | [[log]] | [[hot]] | [[dashboard]] | [[Wiki Map]] | [[getting-started]]

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

---

## Concepts

CUBRID:
- [[Query Processing Pipeline]] — SQL → lexer → parser → name resolution → semantic check → XASL → execute (status: developing)
- [[Build Modes (SERVER SA CS)]] — same source, three binaries via preprocessor guards (status: developing)
- [[Memory Management Conventions]] — `free_and_init`, `db_private_alloc`, `parser_alloc`; no RAII (status: developing)
- [[Error Handling Convention]] — C-style codes, six-place new-error-code rule (status: developing)
- [[Code Style Conventions]] — CI-enforced formatting & naming (status: developing)

LLM Wiki (legacy seed):
- [[LLM Wiki Pattern]] — persistent, compounding knowledge base pattern (status: mature)
- [[Hot Cache]] — ~500-word session context file (status: mature)
- [[Compounding Knowledge]] — why wikis grow more valuable than RAG (status: mature)
- [[cherry-picks]] — prioritized feature backlog (status: current)

---

## Entities

CUBRID:
- [[CUBRID]] — open-source C/C++17 RDBMS with Java PL engine; Apache 2.0; v11.5.x (status: developing)

Legacy seed:
- [[Andrej Karpathy]] — AI researcher, creator of the LLM Wiki pattern, former Tesla AI director (status: developing)
- [[Ar9av-obsidian-wiki]] — multi-agent compatible LLM Wiki plugin; delta tracking manifest (status: current)
- [[Nexus-claudesidian-mcp]] — native Obsidian plugin + MCP bridge; workspace memory, task management (status: current)
- [[ballred-obsidian-claude-pkm]] — goal cascade PKM; auto-commit hooks, /adopt command (status: current)
- [[rvk7895-llm-knowledge-bases]] — 3-depth query system, Marp slides, parallel deep research (status: current)
- [[kepano-obsidian-skills]] — official skills from Obsidian creator; defuddle, obsidian-bases (status: current)
- [[Claudian-YishenTu]] — native Obsidian plugin embedding Claude Code; plan mode, @mention (status: current)

---

## Sources

- [[claude-obsidian-ecosystem-research]] — 2026-04-08 | web research across 16+ repos | 8 wiki pages created

---

## Questions

- [[How does the LLM Wiki pattern work]] — how the pattern works and why it outperforms RAG at human scale (status: developing)

---

## Comparisons

- [[Wiki vs RAG]] — when to use a wiki knowledge base versus RAG; verdict: wiki wins at <1000 pages
- [[claude-obsidian-ecosystem]] — feature matrix of 16+ Claude+Obsidian projects; where claude-obsidian wins and gaps

---

## Domains

<!-- Add domain entries here after scaffold -->
