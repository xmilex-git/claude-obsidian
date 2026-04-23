---
type: component
parent_module: "[[modules/src|src]]"
path: "src/parser/"
status: active
purpose: "SQL text → PT_NODE parse tree → name resolution → semantic check → type inference → XASL plan; bison + flex frontend, client-side only"
key_files:
  - "csql_grammar.y (bison, ~646KB — all SQL syntax)"
  - "csql_lexer.l (flex — tokenisation)"
  - "parse_tree.h (PT_NODE, PT_NODE_TYPE, PT_TYPE_ENUM, PARSER_CONTEXT — the central header)"
  - "parse_tree_cl.c (parser_create_parser, parser_new_node, walker infrastructure)"
  - "name_resolution.c (identifier → schema-object binding)"
  - "semantic_check.c (semantic validation, union compatibility, view cyclic refs)"
  - "type_checking.c (expression type inference, function signatures)"
  - "xasl_generation.c (PT_NODE tree → XASL_NODE tree)"
  - "view_transform.c (mq_translate — view/method inlining)"
  - "compile.c (pt_compile — orchestration)"
  - "show_meta.c (SHOW statement metadata registry)"
  - "method_transform.c (method call transformation)"
  - "parser_allocator.hpp (C++ block_allocator wrapping parser_alloc)"
  - "parse_type.hpp (pt_arg_type, pt_generic_type_enum — function type system)"
  - "double_byte_support.c (multi-byte char handling in lexer)"
public_api:
  - "parser_create_parser() → PARSER_CONTEXT*"
  - "parser_parse_string(parser, buf) → PT_NODE**"
  - "parser_new_node(parser, PT_NODE_TYPE) → PT_NODE*"
  - "parser_walk_tree(parser, node, pre_fn, pre_arg, post_fn, post_arg) → PT_NODE*"
  - "parser_copy_tree / parser_free_tree / parser_free_parser"
  - "pt_compile(parser, statement) → PT_NODE*"
  - "pt_semantic_check(parser, statement) → PT_NODE*"
  - "pt_semantic_type(parser, tree, sc_info) → PT_NODE*"
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
  - "[[Memory Management Conventions]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/name-resolution|name-resolution]]"
  - "[[components/semantic-check|semantic-check]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[components/view-transform|view-transform]]"
  - "[[components/parser-allocator|parser-allocator]]"
  - "[[components/show-meta|show-meta]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/parser/` — SQL Parser & Frontend

Translates SQL text into a validated parse tree (`PT_NODE`), resolves all identifiers against the schema, infers types, and emits an `XASL_NODE` plan handed to the [[components/xasl|XASL executor]] on the server.

## Pipeline (this module's portion)

```
SQL text
  ──► csql_lexer.l          tokenise (Flex)
  ──► csql_grammar.y        shift/reduce → PT_NODE tree (Bison)
  ──► name_resolution.c     bind identifiers to schema objects
  ──► semantic_check.c      validate structure & semantics
  ──► type_checking.c       infer expression types
  ──► view_transform.c      inline views / rewrite method calls   (mq_translate)
  ──► xasl_generation.c     emit XASL_NODE tree
  ──► [serialised → server] xasl_to_stream.c → stream_to_xasl.c
```

See [[Query Processing Pipeline]] for the full path including server-side execution.

## Side of the wire

> [!key-insight] Client-side only
> Every file in `src/parser/` is guarded by `#if defined(SERVER_MODE) #error`. The server never links this code. It only receives a serialised [[components/xasl|XASL]] byte stream.

## Sub-components

| Sub-page | What it covers |
|----------|---------------|
| [[components/parse-tree\|parse-tree]] | `PT_NODE` union structure, `PT_NODE_TYPE` enum, `PARSER_CONTEXT`, traversal |
| [[components/name-resolution\|name-resolution]] | `SCOPES` stack, `pt_bind_names`, resolution algorithm |
| [[components/semantic-check\|semantic-check]] | `pt_semantic_check`, type compatibility, union check |
| [[components/xasl-generation\|xasl-generation]] | `SYMBOL_INFO`, `TABLE_INFO`, `pt_to_regu_variable`, XASL emission |
| [[components/view-transform\|view-transform]] | `mq_translate`, updatability, `mq_rewrite_*` family |
| [[components/parser-allocator\|parser-allocator]] | `parser_block_allocator`, lifetime model |
| [[components/show-meta\|show-meta]] | `SHOWSTMT_METADATA`, DBA-only guard, semantic-check hook |

## Key data structures

### PT_NODE (parse_tree.h)

The universal parse-tree node — a tagged union of ~70 statement/expression types.

```c
struct parser_node          // typedef'd as PT_NODE
{
  PT_NODE_TYPE  node_type;       // discriminant: PT_SELECT, PT_NAME, PT_EXPR, …
  PT_TYPE_ENUM  type_enum;       // resolved SQL type after type_checking
  PT_NODE      *next;            // sibling list link (NULL-terminated)
  PT_NODE      *or_next;         // DNF predicate list link
  PT_NODE      *data_type;       // for complex/parameterized types
  XASL_ID      *xasl_id;         // filled by xasl_generation
  const char   *alias_print;     // column alias text
  int           line_number;
  int           column_number;
  void         *etc;             // application scratch pointer
  UINTPTR       spec_ident;      // entity-spec equivalence class id
  struct { unsigned recompile:1; unsigned cannot_prepare:1; … } flag;
  PT_STATEMENT_INFO info;        // UNION of per-node-type info structs
};
```

`info` is a large anonymous union; the correct member depends on `node_type`:

| `node_type` | `info` member | Key sub-fields |
|-------------|--------------|---------------|
| `PT_SELECT` | `info.query` | `from`, `where`, `group_by`, `having`, `order_by`, `select_list` |
| `PT_NAME` | `info.name` | `original`, `resolved`, `spec_id`, `db_object`, `meta_class` |
| `PT_EXPR` | `info.expr` | `op` (`PT_EQ`, `PT_AND`, …), `arg1`, `arg2`, `arg3` |
| `PT_VALUE` | `info.value` | `data_value` (union), `db_value`, `text` |
| `PT_SPEC` | `info.spec` | `entity_name`, `derived_table`, `cte_pointer`, `id`, `join_type`, `flag` |
| `PT_FUNCTION` | `info.function` | `function_type`, `arg_list`, `analytic` |
| `PT_INSERT/UPDATE/DELETE/MERGE` | `info.insert` etc. | statement-specific |

> [!warning] Wrong info member = UB
> Accessing `node->info.select` when `node_type == PT_EXPR` is undefined behaviour. Always check `node_type` first, or use the typed accessor macros (`PT_EXPR_OP(n)`, `PT_NAME_ORIGINAL(n)`, etc.).

### PARSER_CONTEXT (parse_tree.h)

Owning context for a parse session. Holds the input buffer, error list, host-variable array, query plan trace, authorisation save, and the `COMPILE_CONTEXT`. All nodes allocated via `parser_alloc(parser, size)` are freed when `parser_free_parser(parser)` is called — there is no per-node `free`.

### PT_TYPE_ENUM (parse_tree.h)

Discriminant for SQL types attached to every `PT_NODE`. Starts at 1000 to avoid clashes with `PT_NODE_TYPE`. Key values: `PT_TYPE_INTEGER`, `PT_TYPE_VARCHAR`, `PT_TYPE_JSON`, `PT_TYPE_OBJECT`, `PT_TYPE_SET/MULTISET/SEQUENCE`, `PT_TYPE_ENUMERATION`, timezone-aware variants (`PT_TYPE_DATETIMETZ`, `PT_TYPE_TIMESTAMPLTZ`, …).

`PT_TYPE_NONE` (= 1000) means "not resolved yet". `PT_TYPE_MAYBE` means "indeterminate at compile time" (e.g. a host variable whose type is not yet known).

### pt_arg_type (parse_type.hpp)

Used by `type_checking.c` to describe allowed argument types for built-in operators. A discriminated union of:
- `NORMAL` — exact `PT_TYPE_ENUM`
- `GENERIC` — `pt_generic_type_enum` (e.g. `PT_GENERIC_TYPE_NUMBER`, `PT_GENERIC_TYPE_STRING_VARYING`, `PT_GENERIC_TYPE_JSON_DOC`)
- `INDEX` — "same type as argument N"

## The Bison grammar

> [!warning] `csql_grammar.y` is 646 KB — one of the largest production Bison grammars in any open-source project
> - `YYMAXDEPTH` is set to 1,000,000 (the grammar is deeply recursive).
> - Bison regeneration is slow and fragile; check `y.output` for shift/reduce conflicts after any edit.
> - `PT_TYPE_NCHAR` / `PT_TYPE_VARNCHAR` are deprecated but kept as aliases for `PT_TYPE_CHAR` / `PT_TYPE_VARCHAR` via `#define` in the grammar preamble.
> - Container helper types (`container_2` through `container_11`) group multiple `PT_NODE *` members into a single Bison `$$` value to pass sub-trees up the rule stack.
> - Password offsets (`pwd_info`) are tracked inside grammar actions for security scrubbing via `pt_add_password_offset`.

Grammar-level state variables (e.g. `parser_instnum_check`, `parser_prior_check`, `parser_connectbyroot_check`) are static globals in `csql_grammar.y` toggled by grammar actions to enforce context-sensitive rules (e.g. `ROWNUM` only in `WHERE`/`SELECT` list).

## Compilation orchestration (`compile.c`)

`pt_compile(parser, statement)` drives the full front-end pipeline for one statement:

1. `pt_resolve_names` (name_resolution) — bind `PT_NAME` nodes to `DB_OBJECT *`
2. `pt_semantic_check` (semantic_check) — structural/semantic validation
3. `pt_semantic_type` (type_checking) — type inference
4. `mq_translate` (view_transform) — view inlining, method call rewrite
5. `xasl_generate_statement` (xasl_generation) — emit `XASL_NODE *`

## Memory model

```c
PARSER_CONTEXT *p = parser_create_parser();
// All allocations go into p's internal arena:
void *buf = parser_alloc(p, 256);          // never call free() on this
char *s   = pt_append_string(p, old, new); // returns parser_alloc'd copy
PT_NODE *n = parser_new_node(p, PT_SELECT);
// Entire tree is freed at once:
parser_free_parser(p);   // releases all nodes and strings
```

`parser_block_allocator` (C++ wrapper in `parser_allocator.hpp`) adapts the same arena for use with `cubmem::block_allocator` abstractions.

See [[Memory Management Conventions]] and [[components/parser-allocator|parser-allocator]].

## Tree traversal

Always use `parser_walk_tree`; never write manual recursion:

```c
// Walk pre-order; return modified node from callback.
// Set *continue_walk to PT_STOP_WALK / PT_LEAF_WALK / PT_LIST_WALK.
PT_NODE *my_pre(PARSER_CONTEXT *p, PT_NODE *node, void *arg, int *continue_walk);
PT_NODE *my_post(PARSER_CONTEXT *p, PT_NODE *node, void *arg, int *continue_walk);

parser_walk_tree(parser, root, my_pre, &my_arg, my_post, &my_arg);
```

`parser_walk_leaves` visits children but not the root itself. `pt_continue_walk` is the no-op pass-through callback.

## Error model

Errors are attached to the `PARSER_CONTEXT` (not thrown):

```c
PT_ERROR(parser, node, "message");          // formatted
PT_ERRORm(parser, node, setNo, msgNo);      // message catalogue
PT_WARNING(parser, node, "message");
```

`PT_SET_JMP_ENV(parser)` / `PT_CLEAR_JMP_ENV(parser)` wrap `setjmp`/`longjmp` for OOM recovery: on allocation failure, `parser_alloc` calls `longjmp`, the handler records the error and returns `NULL`. Callers must set up the JMP environment before any allocation-heavy work.

`pt_report_to_ersys(parser, error_type)` converts accumulated parser errors into CUBRID `er_set` error codes for the rest of the engine.

## Adding a built-in SQL function (cross-module recipe)

Per `AGENTS.md`, a new built-in spans four files across two modules:

1. **`csql_grammar.y`** — add keyword token + grammar rule that constructs `PT_FUNCTION` or `PT_EXPR` node
2. **`type_checking.c`** — add case to the operator/function signature table (`pt_arg_type` arrays)
3. **`xasl_generation.c`** — add case to `pt_to_regu_variable` or the function dispatch table
4. **`src/query/`** — add server-side evaluation in `fetch_func_value` or equivalent

## Hint system

`PT_HINT_ENUM` is a 64-bit bitmask stored in `PT_NODE.info.query.hint`. Hints are parsed in the grammar, resolved in name_resolution (table names in `USE_NL`, etc.), and consumed by the optimizer via `QO_PLAN`. Key hints affecting the parser module itself:

| Hint flag | Effect |
|-----------|--------|
| `PT_HINT_RECOMPILE` | Bypass plan cache on next execute |
| `PT_HINT_NO_MERGE` | Suppress view merge in `mq_translate` |
| `PT_HINT_INLINE_CTE` | Force CTE inlining |
| `PT_HINT_MATERIALIZE_CTE` | Force CTE materialisation |
| `PT_HINT_NO_SUBQUERY_CACHE` | Disable subquery result cache |

## Identifier comparison convention

All identifier matching in this module uses `intl_identifier_casecmp()` — case-insensitive, locale-aware. Never use `strcmp` directly for schema names.

## Gotchas

- **`csql_grammar.y` backup files** — `.c~`, `.cpp.orig` may exist in tree; ignore them; do not delete without checking git.
- **`PT_TYPE_NCHAR` / `PT_TYPE_VARNCHAR` deprecated** — grammar maps them to `PT_TYPE_CHAR`/`PT_TYPE_VARCHAR` via `#define` in the grammar preamble. Do not add new code paths for NCHAR.
- **No `THREAD_ENTRY *`** — parser runs client-side; server-thread APIs are unavailable.
- **`spec_ident` vs `spec_id`** — `PT_SPEC` uses `info.spec.id` (UINTPTR) as a unique identity key; `PT_NAME` uses `info.name.spec_id` to cross-reference. These must match after name resolution or XASL generation will silently produce wrong joins.
- **CTE pointer nodes** — `PT_WITH_CLAUSE` references `PT_CTE` nodes; cyclic CTE detection (`pt_check_cyclic_reference_in_view_spec`) happens in semantic_check, not name_resolution.

## Related

- Parent: [[modules/src|src]]
- [[components/parse-tree|parse-tree]] — PT_NODE deep dive
- [[components/name-resolution|name-resolution]] — SCOPES stack algorithm
- [[components/semantic-check|semantic-check]] — semantic_check + type_checking
- [[components/xasl-generation|xasl-generation]] — how PT_NODE becomes XASL_NODE
- [[components/view-transform|view-transform]] — mq_translate
- [[components/parser-allocator|parser-allocator]] — arena lifecycle
- [[components/show-meta|show-meta]] — SHOW statement registry
- [[components/optimizer|optimizer]] — consumes QO_PLAN from xasl_generation
- [[Query Processing Pipeline]]
- [[Memory Management Conventions]]
- Source: [[sources/cubrid-src-parser|cubrid-src-parser]]
