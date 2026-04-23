---
type: meta
title: "Hot Cache"
updated: 2026-04-23T19:40:00
tags:
  - meta
  - hot
status: active
---

# Recent Context

## Last Updated
2026-04-23. Scaffolded Mode B (GitHub / codebase) overlay on top of the existing claude-obsidian seed vault, scoped to documenting CUBRID source tree at `/Users/song/DEV/cubrid/`.

## Key Recent Facts
- Primary scope shift: this vault is now CUBRID architecture documentation (C/C++ RDBMS). Secondary: original claude-obsidian seed pages retained as meta-docs.
- Three-process-group topology expected: client (CCI/JDBC) → broker → CAS + DB server.
- Submodules: cubrid-cci, cubrid-jdbc, cubridmanager.
- Build system: CMake.

## Recent Changes
- Created folders: `wiki/modules/`, `wiki/components/`, `wiki/decisions/`, `wiki/dependencies/`, `wiki/flows/`
- Created hub pages: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Dependency Graph]], [[Key Decisions]]
- Seeded [[modules/_index|Modules index]] with all 24 CUBRID top-level directories
- Added Mode B templates: module, component, decision, dependency, flow
- Updated vault [[CLAUDE]] with CUBRID scope and Mode B conventions
- Logged scaffold event in [[log]]

## Active Threads
- Awaiting user confirmation to start ingesting CUBRID sources
- Planned ingest order: AGENTS.md → CMakeLists.txt → build.sh → per-module (broker, cs, src, pl_engine, cm_common, conf, 3rdparty)
- MCP `obsidian-vault` configured (filesystem, @bitbonsai/mcpvault). Effective after Claude Code restart.
