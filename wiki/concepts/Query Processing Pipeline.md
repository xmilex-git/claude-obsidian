---
type: concept
title: "Query Processing Pipeline"
status: developing
tags:
  - concept
  - cubrid
  - query
  - flow
related:
  - "[[CUBRID]]"
  - "[[components/parser|parser]]"
  - "[[components/optimizer|optimizer]]"
  - "[[Data Flow]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# Query Processing Pipeline

End-to-end path from SQL text to executed result in CUBRID.

```
SQL text
  → Lexer            (csql_lexer.l)
  → Parser           (csql_grammar.y → PT_NODE tree)
  → Name Resolution  (name_resolution.c)
  → Semantic Check   (semantic_check.c)
  → XASL Generation  (xasl_generation.c → XASL_NODE tree)
  → Serialization    (xasl_to_stream.c)        ← client side
                                              ─── network ───
  → Deserialization  (stream_to_xasl.c)        ← server side
  → Execution        (query_executor.c)
      └── scans      (scan_manager.c)
```

## Side-of-line

> [!key-insight] Parser + optimizer are client-side
> Everything from lexer through XASL generation runs in the **client** process (`#if !defined(SERVER_MODE)`). The server only sees the serialized XASL plan. This is the fundamental reason XASL exists as a serializable IR.

## Key data structures along the path

| Stage | Structure | Header |
|-------|-----------|--------|
| Parse tree | `PT_NODE` | `src/parser/parse_tree.h` |
| Plan IR | `XASL_NODE` | `src/query/xasl.h` + `src/xasl/` |
| Value container | `DB_VALUE` | `src/compat/dbtype_def.h` |

## Entry points for common modifications

- **Add SQL syntax** → `src/parser/csql_grammar.y` (646 KB bison grammar)
- **Add a built-in function** → spans 4 modules: parser → type_checking → xasl_generation → query/
- **Fix execution** → `qexec_execute_mainblock()` in `src/query/query_executor.c`

## Related

- Components: [[components/parser]], [[components/optimizer]], [[components/query|query]], [[components/xasl|xasl]]
- Hub: [[Data Flow]]
- Source: [[cubrid-AGENTS]]
