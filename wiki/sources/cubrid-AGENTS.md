---
type: source
title: "CUBRID AGENTS.md (project guide)"
source_type: file
source_path: ".raw/cubrid/AGENTS.md"
ingested: 2026-04-23
hash: 946ec2715e851332b3d414853aebe1d4
status: summarized
tags:
  - source
  - cubrid
  - guide
related:
  - "[[CUBRID]]"
  - "[[Architecture Overview]]"
  - "[[Query Processing Pipeline]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Memory Management Conventions]]"
  - "[[Error Handling Convention]]"
  - "[[Code Style Conventions]]"
  - "[[modules/src|src module]]"
  - "[[modules/broker|broker module]]"
  - "[[modules/pl_engine|pl_engine module]]"
  - "[[modules/unit_tests|unit_tests module]]"
created: 2026-04-23
updated: 2026-04-23
---

# CUBRID AGENTS.md — Project Guide

Top-level orientation document for the CUBRID source tree. Targeted at engineers (and AI agents) navigating the codebase. ~216 lines.

## What it covers

1. **Overview** — CUBRID is an open-source C/C++17 RDBMS with a Java PL engine. Apache 2.0. v11.5.x.
2. **Structure** — annotated tree of every top-level directory and `src/` subdirectory, with one-line purpose each.
3. **Where to look** — task → file/function lookup table for common modifications.
4. **Query processing pipeline** — SQL text → lexer → parser → name resolution → semantic check → XASL gen → serialize → deserialize → execute.
5. **Build commands & options** — `build.sh` flags, CMake presets.
6. **Build modes** — `SERVER_MODE` / `SA_MODE` / `CS_MODE` preprocessor guards.
7. **Code style** — formatting (CI-enforced), naming, includes (with `memory_wrapper.hpp` last-include rule).
8. **Error handling** — `er_set` model, 6-place new-error-code procedure.
9. **Memory management** — `free_and_init`, `db_private_alloc`, `parser_alloc`.
10. **Key data structures** — PT_NODE, XASL_NODE, DB_VALUE, PAGE_BUFFER, LOCK_RESOURCE, LOG_RECORD_HEADER, MVCC_TRANS_STATUS.
11. **CI** — GitHub Actions (lint), Jenkins (primary build), CircleCI (SQL/shell tests).
12. **Anti-patterns & gotchas** — 9 anti-patterns + 7 gotchas specific to this codebase.
13. **Important references** — 4 must-read headers/docs.

## Highest-leverage facts

> [!key-insight] `.c` compiled as C++17
> All `.c` files are intentionally compiled as C++17 (see `c_to_cpp.sh`). This is a deliberate decision, not a migration in progress.

> [!key-insight] Same source, three binaries
> The same source compiles to `cub_server` (server), `cubridsa` (standalone in-process), and `cubridcs` (client lib) via `SERVER_MODE` / `SA_MODE` / `CS_MODE` guards. **Parser & optimizer are client-side only.**

> [!key-insight] Two `broker/` directories
> Top-level `broker/` is a CMake target + configs. `src/broker/` is the actual implementation (CAS connection broker). They are NOT the same and not interchangeable.

> [!key-insight] Adding an error code touches 6 files
> `error_code.h`, `dbi_compat.h`, `cubrid.msg` (en + ko), `ER_LAST_ERROR`, and `cubrid-cci/base_error_code.h` if client-facing. Forgetting any one breaks the build or i18n.

> [!key-insight] `csql_grammar.y` is 646 KB
> The bison grammar is gigantic. Edits need extreme care; regeneration is slow.

## Pages this source produced

- Entity: [[CUBRID]]
- Concepts: [[Query Processing Pipeline]], [[Build Modes (SERVER SA CS)]], [[Memory Management Conventions]], [[Error Handling Convention]], [[Code Style Conventions]]
- Modules: [[modules/src|src]], [[modules/broker|broker]], [[modules/pl_engine|pl_engine]], [[modules/unit_tests|unit_tests]]
- Components: [[components/parser|parser]], [[components/optimizer|optimizer]], [[components/storage|storage]], [[components/transaction|transaction]]
- Hub updates: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]], [[Key Decisions]]

## Source location

`.raw/cubrid/AGENTS.md` (symlink → `/Users/song/DEV/cubrid/AGENTS.md`). Note: the source tree's `CLAUDE.md` is a symlink to `AGENTS.md`, so both names point to the same file.
