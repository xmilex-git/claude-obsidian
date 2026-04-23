---
type: component
parent_module: "[[components/parser|parser]]"
path: "src/parser/semantic_check.c, src/parser/type_checking.c"
status: active
purpose: "Post-name-resolution semantic validation (structural correctness, view cyclic refs, union compatibility) and expression type inference"
key_files:
  - "semantic_check.c (pt_semantic_check — main driver)"
  - "semantic_check.h (public API, STATEMENT_SET_FOLD enum)"
  - "type_checking.c (pt_semantic_type, pt_eval_function_arg_types)"
  - "parse_type.hpp (pt_arg_type, pt_generic_type_enum)"
public_api:
  - "pt_semantic_check(parser, statement) → PT_NODE*"
  - "pt_semantic_type(parser, tree, sc_info) → PT_NODE*"
  - "pt_check_union_compatibility(parser, node) → PT_NODE*"
  - "pt_check_type_compatibility_of_values_query(parser, node) → PT_NODE*"
  - "pt_check_union_is_foldable(parser, node) → STATEMENT_SET_FOLD"
  - "pt_fold_union(parser, node, fold_as) → PT_NODE*"
  - "pt_semantic_quick_check_node(parser, spec*, node*) → PT_NODE*"
  - "pt_invert(parser, name_expr, result) → PT_NODE*"
  - "pt_check_cast_op(parser, node) → bool"
  - "pt_try_remove_order_by(parser, query)"
tags:
  - component
  - cubrid
  - parser
  - semantic-check
  - type-checking
related:
  - "[[components/parser|parser]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/name-resolution|name-resolution]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# Semantic Check & Type Checking

Two passes that run after [[components/name-resolution|name resolution]] and before [[components/xasl-generation|XASL generation]].

- `semantic_check.c` — structural and semantic validation.
- `type_checking.c` — expression type inference; populates `PT_NODE.type_enum` on every expression node.

Both are called from `pt_compile` (in `compile.c`) and share the `SEMANTIC_CHK_INFO` struct.

## semantic_check.c — what it validates

`pt_semantic_check(parser, statement)` is a second `parser_walk_tree` pass. Key checks:

| Check function | What it does |
|---------------|-------------|
| `pt_check_union_compatibility` | Ensures UNION/INTERSECT/EXCEPT branches have compatible column counts and types; inserts implicit CASTs |
| `pt_check_type_compatibility_of_values_query` | Validates `VALUES (...)` rows against the target column list |
| `pt_check_cyclic_reference_in_view_spec` | Detects circular view references (A→B→A); sets an error and stops walk |
| `pt_check_odku_assignments` | Validates ON DUPLICATE KEY UPDATE assignments; checks for uniqueness conflicts |
| `pt_check_cast_op` | Validates that a CAST target type is reachable from the source type |
| `pt_try_remove_order_by` | Strips ORDER BY from subqueries where it has no semantic effect |
| `pt_check_compatible_node_for_orderby` | Checks ORDER BY column is compatible with SELECT list |
| `pt_insert_entity` | Resolves the implicit entity spec for path expressions |
| `pt_find_class_of_index` | Locates the table owning a named index |
| `pt_invert` | Inverts an assignment expression (for view update translation) |

### Union foldability

`STATEMENT_SET_FOLD` indicates whether a UNION/INTERSECT/EXCEPT can be folded:

```c
typedef enum {
  STATEMENT_SET_FOLD_NOTHING  = 0,  // cannot fold
  STATEMENT_SET_FOLD_AS_NULL,       // fold entire set op to NULL
  STATEMENT_SET_FOLD_AS_ARG1,       // fold to left branch
  STATEMENT_SET_FOLD_AS_ARG2        // fold to right branch
} STATEMENT_SET_FOLD;
```

`pt_check_union_is_foldable` checks whether one branch is always empty (false WHERE, zero-row VALUES). `pt_fold_union` rewrites the tree accordingly.

## type_checking.c — type inference

`pt_semantic_type(parser, tree, sc_info)` runs a `parser_walk_tree` in post-order (bottom-up), setting `type_enum` on each expression node.

### pt_arg_type — function signature system

Each built-in operator/function has an array of `pt_arg_type` entries describing allowed argument types:

```cpp
// From parse_type.hpp:
struct pt_arg_type {
  enum { NORMAL, GENERIC, INDEX } type;
  union {
    PT_TYPE_ENUM type;              // exact type
    pt_generic_type_enum generic;   // family of types
    size_t index;                   // "same type as arg N"
  } val;
};
```

Generic types (`pt_generic_type_enum`):

| Enum | Matches |
|------|---------|
| `PT_GENERIC_TYPE_NUMBER` | Any numeric type |
| `PT_GENERIC_TYPE_STRING` | Any string (CHAR, VARCHAR, BIT, VARBIT) |
| `PT_GENERIC_TYPE_STRING_VARYING` | VARCHAR only |
| `PT_GENERIC_TYPE_CHAR` | VARCHAR or CHAR |
| `PT_GENERIC_TYPE_DISCRETE_NUMBER` | SMALLINT, INTEGER, BIGINT |
| `PT_GENERIC_TYPE_DATE` | DATE, DATETIME, TIMESTAMP |
| `PT_GENERIC_TYPE_DATETIME` | Any date/time type |
| `PT_GENERIC_TYPE_JSON_VAL` | JSON-compatible scalar |
| `PT_GENERIC_TYPE_JSON_DOC` | JSON document type |
| `PT_GENERIC_TYPE_SEQUENCE` | SET, MULTISET, SEQUENCE |
| `PT_GENERIC_TYPE_ANY` | Any type (no restriction) |

### Type coercion

When argument types don't exactly match a signature, `type_checking.c` inserts implicit `PT_CAST` (`PT_EXPR` with `op == PT_CAST`) nodes. The inserted cast is transparent to later passes.

`PT_TYPE_MAYBE` propagates through expressions when a host variable's type is still unknown. The actual type is resolved at execution time by the XASL interpreter.

### Collation inference

String expressions accumulate a `coll_modifier` (stored as `modifier + 1` so 0 = "not set"). When two string operands have different collations, type_checking inserts a COLLATE conversion or raises an error depending on the coercibility rules.

## SEMANTIC_CHK_INFO

Shared state threaded through both passes:

```c
// (inferred from usage in semantic_check.h and compile.c)
typedef struct {
  bool top_level;           // is this the top-level statement?
  bool Oracle_compat;       // Oracle outer-join compat mode
  PT_NODE *where_clause;    // current WHERE being analysed
} SEMANTIC_CHK_INFO;
```

## Interaction with view_transform

`pt_semantic_check` runs before `mq_translate` (view_transform). Some semantic checks are repeated lightly after view inlining to validate the rewritten tree. The `pt_semantic_quick_check_node` function provides a targeted re-check for single nodes without a full tree walk.

## Error reporting

Semantic errors are attached via `PT_ERROR` / `PT_ERRORm` macros (which call `pt_frob_error`). They accumulate in `parser->error_msgs`. After `pt_semantic_check` returns, `parser_has_error(parser)` reports whether any errors were found; the caller in `compile.c` aborts the pipeline if so.

## Related

- Parent: [[components/parser|parser]]
- [[components/parse-tree|parse-tree]] — PT_NODE structure, type_enum field
- [[components/name-resolution|name-resolution]] — previous pass; provides db_object bindings
- [[components/xasl-generation|xasl-generation]] — next pass; consumes type_enum
- [[components/view-transform|view-transform]] — runs after semantic_check; may trigger re-check
