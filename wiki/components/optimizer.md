---
type: component
parent_module: "[[modules/src|src]]"
path: "src/optimizer/"
status: active
purpose: "Cost-based query planning"
key_files: []
public_api: []
tags:
  - component
  - cubrid
  - optimizer
  - query
related:
  - "[[modules/src|src]]"
  - "[[Query Processing Pipeline]]"
  - "[[components/parser|parser]]"
  - "[[components/query|query]]"
  - "[[components/xasl|xasl]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/optimizer/` — Cost-Based Query Planner

Selects join orders, access paths, and physical operators based on cost estimates. Output feeds [[components/xasl|XASL]] generation in [[components/parser]].

## Side of the wire

> [!key-insight] Client-side
> Like the parser, the optimizer runs on the **client** (`#if !defined(SERVER_MODE)`). The server receives only the chosen plan as serialized XASL.

## Inputs / outputs

- **Input:** annotated `PT_NODE` tree from [[components/parser]] (after name resolution + semantic check)
- **Output:** decisions consumed by `xasl_generation.c` to build the `XASL_NODE` plan

## Notes

Detail will be added on a deeper ingest of the optimizer source. AGENTS.md only lists this directory at one-line granularity.

## Related

- Parent: [[modules/src|src]]
- [[Query Processing Pipeline]]
- Source: [[cubrid-AGENTS]]
