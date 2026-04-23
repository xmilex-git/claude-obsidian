---
type: overview
title: "Wiki Overview"
created: 2026-04-07
updated: 2026-04-23
tags:
  - meta
  - overview
status: active
related:
  - "[[index]]"
  - "[[hot]]"
  - "[[log]]"
  - "[[dashboard]]"
  - "[[CUBRID]]"
  - "[[Architecture Overview]]"
  - "[[LLM Wiki Pattern]]"
sources:
  - "[[cubrid-AGENTS]]"
---

# Wiki Overview

Navigation: [[index]] | [[hot]] | [[log]] | [[dashboard]] | [[Wiki Map]]

---

## Purpose

Primary scope: **Documenting the CUBRID relational database source tree** (Mode B — GitHub / codebase wiki). The vault maps CUBRID's subsystems, data structures, data flows, conventions, and design decisions from the C/C++17 source at `/Users/song/DEV/cubrid/`.

Secondary scope: A small cluster of legacy seed pages about the LLM Wiki pattern itself (how this vault works) is retained as meta-documentation.

---

## Current State

- **Total wiki pages:** ~209
- **Components:** 116 (one per CUBRID subsystem)
- **Modules:** 10 (top-level source directories)
- **Sources ingested:** 32 (21 CUBRID source-tree ingests + legacy seed sources)
- **Concepts:** 10 (5 CUBRID conventions + 5 legacy seed)
- **Flows:** 2 composed + `_index` (DML / DDL execution paths)
- **Dependencies:** 11 (bundled 3rd-party libraries)
- **Entities:** 9 ([[CUBRID]] + legacy)
- **Last major activity:** 2026-04-23 — full src/ tree + resource dirs (3rdparty, locales, timezones, msg, contrib) ingested; DML + DDL flow pages composed; wiki linted.

---

## Hub Pages (start here)

- [[Architecture Overview]] — 3-tier topology, client-side vs server-side split, build modes
- [[Tech Stack]] — languages, build, bundled libs, CI
- [[Data Flow]] — query path + links to flow pages
- [[Dependency Graph]] — internal + external deps
- [[Key Decisions]] — cross-cutting design choices

## Flow Pages

- [[flows/dml-execution-path|DML execution path]] — INSERT/UPDATE/DELETE/MERGE end-to-end
- [[flows/ddl-execution-path|DDL execution path]] — CREATE/ALTER/DROP/GRANT end-to-end

## Sub-Indexes

- [[modules/_index|Modules]] — top-level source dirs (`src/`, `broker/`, `cs/`, `sa/`, `cubrid/`, `pl_engine/`, …)
- [[components/_index|Components]] — subsystems grouped by module (23 sections)
- [[sources/_index|Sources]] — ingest catalog
- [[decisions/_index|Decisions]] — ADRs (mostly empty — populated by lint follow-ups)
- [[dependencies/_index|Dependencies]] — 3rd-party libraries
- [[flows/_index|Flows]] — request paths and lifecycles
- [[concepts/_index|Concepts]] — conventions and patterns
- [[entities/_index|Entities]] — products, orgs, people

---

## Key CUBRID Facts (at a glance)

- Open-source C/C++17 RDBMS + Java PL engine for stored procedures. Apache 2.0. v11.5.x.
- Same source → 3 binaries via `SERVER_MODE` / `SA_MODE` / `CS_MODE`. See [[Build Modes (SERVER SA CS)]].
- 3-tier topology: client (CCI/JDBC) → broker → CAS workers → DB server.
- Parser + optimizer run **client-side**; only serialized [[components/xasl|XASL]] crosses the wire.
- C error model (`er_set`) in C++ code; no exceptions, no RAII. See [[Error Handling Convention]], [[Memory Management Conventions]].

---

## Canvases

- [[Wiki Map]] — central knowledge graph canvas
- [[claude-obsidian-presentation]] — legacy seed presentation
- [[AI Marketing Hub Cover Images Canvas]] — legacy seed brand assets

---

## Key Themes (inherited from the LLM Wiki pattern)

**Knowledge compounds.** Unlike RAG, the wiki pre-compiles synthesis. Cross-references are already there. Contradictions are flagged (see callouts in [[components/transaction]], [[components/dbi-compat]]). Every ingest enriches existing pages rather than adding isolated chunks.

**The hot cache is the force multiplier.** [[hot]] (~500 words) captures recent context. New sessions start with full context at minimal token cost.

**Obsidian is the IDE, Claude is the programmer.** The graph view shows what's connected. The human curates sources and asks questions. Claude writes and maintains everything else.
