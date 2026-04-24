---
type: meta
title: "Concepts Index"
created: 2026-04-07
updated: 2026-04-24
tags:
  - meta
  - index
  - concept
domain: knowledge-management
status: evergreen
related:
  - "[[index]]"
  - "[[dashboard]]"
  - "[[Wiki Map.canvas]]"
---

# Concepts Index

Navigation: [[index]] | [[entities/_index|Entities]] | [[sources/_index|Sources]]

CUBRID conventions, patterns, and cross-cutting ideas extracted from the source tree.

---

## CUBRID Conventions

- [[Query Processing Pipeline]] — SQL → lexer → parser → name resolution → semantic check → XASL → execute
- [[Build Modes (SERVER SA CS)]] — same source, three binaries via preprocessor guards
- [[Memory Management Conventions]] — `free_and_init`, `db_private_alloc`, `parser_alloc`; no RAII in C++ code
- [[Error Handling Convention]] — C-style codes, six-place new-error-code rule
- [[Code Style Conventions]] — CI-enforced formatting & naming

---

## Legacy seed (LLM Wiki pattern itself)

Moved to `_legacy/` — see [[_legacy/_index|Legacy Seed Index]].

## Add new CUBRID concepts here as they are extracted.
