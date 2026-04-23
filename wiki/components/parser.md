---
type: component
parent_module: "[[modules/src|src]]"
path: "src/parser/"
status: active
purpose: "SQL → parse tree (PT_NODE) → XASL generation; bison + flex"
key_files:
  - "csql_grammar.y (bison, ~646KB)"
  - "csql_lexer.l (flex)"
  - "parse_tree.h (PT_NODE definitions)"
  - "name_resolution.c"
  - "semantic_check.c"
  - "xasl_generation.c"
  - "type_checking.c"
public_api: []
tags:
  - component
  - cubrid
  - parser
  - query
related:
  - "[[modules/src|src]]"
  - "[[Query Processing Pipeline]]"
  - "[[components/optimizer|optimizer]]"
  - "[[components/xasl|xasl]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/parser/` — SQL Parser & Frontend

Translates SQL text into the parse tree (`PT_NODE`), validates names and types, then emits XASL plans for the optimizer/executor.

## Stages it owns

```
SQL text
  → Lexer (csql_lexer.l)
  → Parser (csql_grammar.y → PT_NODE)
  → Name Resolution (name_resolution.c)
  → Semantic Check (semantic_check.c)
  → XASL Generation (xasl_generation.c → XASL_NODE)
```

See [[Query Processing Pipeline]] for the full path including server-side execution.

## Side of the wire

> [!key-insight] Client-side
> All parser work runs on the **client**, guarded by `#if !defined(SERVER_MODE)`. The server only sees serialized [[components/xasl|XASL]].

## Key data structure

`PT_NODE` (defined in `parse_tree.h`) — union-based, linked-list parse tree node. Almost every parser function takes a `PT_NODE *` and a `PARSER_CONTEXT *`.

## Modifying the grammar

> [!warning] `csql_grammar.y` is 646 KB
> Bison regeneration is slow. Edits should be minimal and well-tested. Backup files (`.c~`, `.cpp.orig`) may exist in the tree — ignore them.

## Adding a built-in SQL function

Per [[cubrid-AGENTS|AGENTS.md]], a built-in function spans **4 modules**:

```
parser → type_checking → xasl_generation → query/
```

So a new function changes this component **and** [[components/query|query]].

## Memory

Parser-lifetime allocations use `parser_alloc(parser, len)` — see [[Memory Management Conventions]].

## Related

- Parent: [[modules/src|src]]
- Source: [[cubrid-AGENTS]]
