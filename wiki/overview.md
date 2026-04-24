---
type: overview
title: "Wiki Overview"
created: 2026-04-07
updated: 2026-04-24
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
  - "[[_legacy/_index]]"
sources:
  - "[[cubrid-AGENTS]]"
---

# Wiki Overview

Navigation: [[index]] | [[hot]] | [[log]] | [[dashboard]] | [[Wiki Map.canvas|Wiki Map]]

---

## Purpose

Primary scope: **Documenting the CUBRID relational database source tree** (Mode B — GitHub / codebase wiki). The vault maps CUBRID's subsystems, data structures, data flows, conventions, and design decisions from the C/C++17 source at `~/dev/cubrid/`.

Secondary scope: Legacy seed pages about the LLM Wiki pattern and claude-obsidian plugin history are archived under `wiki/_legacy/` — see [[_legacy/_index|Legacy Seed Index]].

---

## Current State

- **Total wiki pages:** ~246
- **Components:** 111 (one per CUBRID subsystem)
- **Modules:** 10 (top-level source directories)
- **Sources ingested:** 27 (21 CUBRID source-tree ingests + a few composed query-family pages)
- **Concepts:** 5 (CUBRID conventions only; LLM Wiki pattern pages in `_legacy/`)
- **Flows:** 2 composed + `_index` (DML / DDL execution paths)
- **Dependencies:** 10 (bundled 3rd-party libraries) + `_index`
- **Entities:** 1 ([[CUBRID]]); legacy seed entities in `_legacy/entities/`
- **Last major activity:** 2026-04-24 — legacy seed archived under `_legacy/`; lint report 2026-04-24 filed.

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
- [[concepts/_index|Concepts]] — CUBRID conventions and patterns
- [[entities/_index|Entities]] — CUBRID product entity

---

## Key CUBRID Facts (at a glance)

- Open-source C/C++17 RDBMS + Java PL engine for stored procedures. Apache 2.0. v11.5.x.
- Same source → 3 binaries via `SERVER_MODE` / `SA_MODE` / `CS_MODE`. See [[Build Modes (SERVER SA CS)]].
- 3-tier topology: client (CCI/JDBC) → broker → CAS workers → DB server.
- Parser + optimizer run **client-side**; only serialized [[components/xasl|XASL]] crosses the wire.
- C error model (`er_set`) in C++ code; no exceptions, no RAII. See [[Error Handling Convention]], [[Memory Management Conventions]].

---

## Canvases

- [[Wiki Map.canvas|Wiki Map]] — central knowledge graph canvas

---

## How the Vault Works

Knowledge compounds across ingests. The wiki pre-compiles synthesis and cross-references; contradictions are flagged with `> [!contradiction]` callouts (see [[components/transaction]], [[components/dbi-compat]]). Every ingest enriches existing pages rather than adding isolated chunks. The hot cache ([[hot]], ~500 words) captures recent context and loads automatically at session start.
