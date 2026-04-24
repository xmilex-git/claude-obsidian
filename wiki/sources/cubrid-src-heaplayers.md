---
created: 2026-04-23
type: source
title: "CUBRID src/heaplayers/ — Embedded Heap Allocator Library"
source_path: "src/heaplayers/"
date_ingested: 2026-04-23
scope: "Directory listing + heaplayers.h header; lea_heap.c header only (opaque 3rd-party body)"
status: complete
tags:
  - source
  - cubrid
  - memory
  - third-party
related:
  - "[[components/heaplayers|heaplayers]]"
  - "[[components/memory-alloc|memory-alloc]]"
---

# Source: `src/heaplayers/`

Light-touch ingest. This directory is a vendored 3rd-party library; implementation files were not read deeply.

## What Was Read

- Directory structure (file list)
- `heaplayers.h` (master include — full)
- `lea_heap.c` (header comment only — body is opaque 3rd-party)
- Project AGENTS.md anti-patterns and CI rules referencing this directory

## Key Facts Captured

- **Library:** Heap Layers by Emery Berger (Apache 2.0), based on PLDI 2001 paper
- **Primary file:** `lea_heap.c` — ~181 KB Doug Lea malloc (dlmalloc) adaptation
- **Namespace:** `HL` (C++)
- **Sub-groups:** `utility/`, `heaps/`, `locks/`, `threads/`, `wrappers/` (each has `all.h`)
- **Engine consumer:** `db_private_alloc` in `src/base/memory_alloc.c` via `HL_HEAPID` opaque handle
- **Activation:** SERVER_MODE only; CS_MODE and SA_MODE bypass this library

## Project Rules

- Do not modify any file in this directory
- Entire directory excluded from CI cppcheck
- `lea_heap.c` is explicitly called out in AGENTS.md as must-not-modify

## Pages Created

- [[components/heaplayers|heaplayers]] — component hub page
