---
status: active
created: 2026-04-23
type: source
title: "CUBRID src/xasl/ — XASL Node Type Definitions"
source_path: "~/dev/cubrid/src/xasl/"
date_ingested: 2026-04-23
ingestion_mode: B (codebase)
tags:
  - source
  - cubrid
  - xasl
  - query
related:
  - "[[components/xasl|xasl]]"
  - "[[components/xasl-stream|xasl-stream]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/xasl-predicate|xasl-predicate]]"
  - "[[components/xasl-aggregate|xasl-aggregate]]"
  - "[[components/xasl-analytic|xasl-analytic]]"
---

# Source: `src/xasl/` — XASL Node Type Definitions

## Files read

| File | Purpose |
|------|---------|
| `src/xasl/AGENTS.md` | Module overview, key files, conventions, gotchas |
| `src/xasl/xasl_predicate.hpp` | `PRED_EXPR` tree, `REL_OP`, `BOOL_OP`, eval-term variants |
| `src/xasl/xasl_aggregate.hpp` | `aggregate_list_node`, `aggregate_accumulator`, percentile/cume info |
| `src/xasl/xasl_analytic.hpp` | `analytic_list_node`, `analytic_eval_type`, per-function runtime state |
| `src/xasl/xasl_stream.hpp` | `stx_build`/`stx_restore` templates, alignment constants, XASL_UNPACK_INFO helpers |
| `src/xasl/xasl_unpack_info.hpp` | `XASL_UNPACK_INFO`, visited-pointer hash table, extra-buffer chain |
| `src/xasl/xasl_sp.hpp` | `SP_TYPE` (stored-procedure invocation node) |
| `src/query/xasl.h` | `XASL_NODE`, `PROC_TYPE`, `XASL_ID`, `XASL_STREAM`, all proc-node subtypes, access specs |
| `src/query/regu_var.hpp` | `regu_variable_node` / `REGU_VARIABLE`, `REGU_DATATYPE`, `ARITH_TYPE`, `FUNCTION_TYPE` |
| `src/query/xasl_to_stream.h` | Client packer API (`xts_map_xasl_to_stream`) |
| `src/query/stream_to_xasl.h` | Server unpacker API (`stx_map_stream_to_xasl`) |

## Key findings

### 1. Split ownership between `src/xasl/` and `src/query/`
`src/xasl/` holds the modular type headers (predicates, aggregates, analytics, stream helpers). The main plan node (`XASL_NODE`) and the most complex expression type (`REGU_VARIABLE`) live in `src/query/xasl.h` and `src/query/regu_var.hpp` respectively. This is a known historical quirk — `regu_var.hpp` cannot move to `src/xasl/` without circular-include surgery.

### 2. Offset-based pointer encoding
The serialisation protocol replaces every pointer with an integer byte offset from the start of the packed body. `stx_restore<T>` resolves offsets back to pointers using a 256-bucket visited-pointer hash table (`XASL_UNPACK_INFO.ptr_blocks`). Shared sub-structures are deserialised exactly once.

### 3. REGU_VARIABLE complexity
`regu_variable_node` is a 17-member union (`REGU_DATATYPE` discriminant) encoding: literals, host variables, heap attribute fetches, list-file positions, arithmetic trees, function calls, subquery links, aggregate links, VALUES lists, CUME_DIST argument lists, and stored-procedure invocations. It is the single most-modified structure in the XASL layer.

### 4. Runtime vs serialised field split
All four major node types (`XASL_NODE`, `AGGREGATE_TYPE`, `ANALYTIC_TYPE`, `REGU_VARIABLE`) follow the same pattern: fields up to a certain point are serialised; everything after is runtime-only server state (accumulators, list-file IDs, scan IDs, stat structs). The split is enforced by `#if defined(SERVER_MODE) || defined(SA_MODE)` guards on runtime-only fields.

### 5. Four-file rule for XASL changes
Adding any field requires: (1) header, (2) `xasl_to_stream.c`, (3) `stream_to_xasl.c`, (4) `xasl_generation.c`. Mismatch between pack and unpack causes silent data corruption or crash at deserialization — there is no version header in the stream.

### 6. `UNPACK_SCALE = 3`
The unpacked XASL tree is approximately 3× larger than the packed stream. Callers must pre-allocate a working arena of at least `3 × stream_size`.

## Pages created

- [[components/xasl|xasl]] — hub component page (satisfied the pre-existing stub link)
- [[components/xasl-stream|xasl-stream]] — serialisation protocol
- [[components/regu-variable|regu-variable]] — REGU_VARIABLE deep dive
- [[components/xasl-predicate|xasl-predicate]] — PRED_EXPR tree
- [[components/xasl-aggregate|xasl-aggregate]] — AGGREGATE_TYPE node
- [[components/xasl-analytic|xasl-analytic]] — ANALYTIC_TYPE / ANALYTIC_EVAL_TYPE
