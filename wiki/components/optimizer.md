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

## Selectivity defaults

`query_planner.c` defines a set of `DEFAULT_*_SELECTIVITY` constants used whenever no statistics-backed estimate is available. These live in the `.c` as file-private `#define`s:

| Constant | Value | Used for |
|---|---|---|
| `DEFAULT_NULL_SELECTIVITY` | 0.01 | `IS NULL` when null stats unavailable |
| `DEFAULT_EXISTS_SELECTIVITY` | 0.1 | `EXISTS (subq)` |
| `DEFAULT_SELECTIVITY` | 0.1 | generic fallback |
| `DEFAULT_EQUAL_SELECTIVITY` | 0.001 | `ATTR = const` |
| `DEFAULT_EQUIJOIN_SELECTIVITY` | 0.001 | `ATTR = ATTR` across relations |
| `DEFAULT_COMP_SELECTIVITY` | 0.1 | `ATTR {<,<=,>,>=} const` |
| `DEFAULT_BETWEEN_SELECTIVITY` | 0.01 | `ATTR BETWEEN a AND b` |
| `DEFAULT_IN_SELECTIVITY` | 0.01 | `ATTR IN (...)` |
| `DEFAULT_RANGE_SELECTIVITY` | 0.1 | composite range terms |

`PRM_ID_LIKE_TERM_SELECTIVITY` (system parameter) drives the default for `LIKE` / `LIKE ESCAPE`. These constants are the bedrock cost-model assumptions — they are calibration-sensitive and rarely touched.

`PRED_CLASS { PC_ATTR, PC_CONST, PC_HOST_VAR, PC_SUBQUERY, PC_SET, PC_OTHER, PC_MULTI_ATTR }` and the file-local `qo_classify` helper partition predicate operands for the selectivity dispatch.

## Inputs / outputs, continued

Detail will be added on a deeper ingest of the optimizer source. AGENTS.md only lists this directory at one-line granularity.

## Related

- Parent: [[modules/src|src]]
- [[Query Processing Pipeline]]
- Source: [[cubrid-AGENTS]]
