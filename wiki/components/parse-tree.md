---
type: component
parent_module: "[[components/parser|parser]]"
path: "src/parser/parse_tree.h, parse_tree_cl.c"
status: active
purpose: "PT_NODE tagged-union parse tree node — the universal intermediate representation between SQL text and XASL"
key_files:
  - "parse_tree.h (PT_NODE struct, PT_NODE_TYPE enum, PT_TYPE_ENUM, PARSER_CONTEXT, all macros)"
  - "parse_tree_cl.c (parser_create_parser, parser_new_node, walker implementation)"
  - "parser.h (public API: parse, walk, copy, free, print functions)"
public_api:
  - "parser_new_node(parser, PT_NODE_TYPE) → PT_NODE*"
  - "parser_walk_tree(parser, node, pre_fn, pre_arg, post_fn, post_arg) → PT_NODE*"
  - "parser_copy_tree / parser_copy_tree_list"
  - "parser_append_node(node, list) → PT_NODE*"
  - "parser_free_tree(parser, tree)"
  - "parser_print_tree(parser, node) → char*"
tags:
  - component
  - cubrid
  - parser
related:
  - "[[components/parser|parser]]"
  - "[[components/name-resolution|name-resolution]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `PT_NODE` — Parse Tree Node

`PT_NODE` (`struct parser_node`) is the single C struct that represents every element of a parsed SQL statement — statements, clauses, expressions, names, values, type descriptors. It is a tagged union discriminated by `node_type`.

## Struct layout (`parse_tree.h`)

```c
struct parser_node            // typedef PT_NODE
{
  /* --- discriminant & location --- */
  PT_NODE_TYPE  node_type;       // what kind of node this is
  int           line_number;     // source line (1-based)
  int           column_number;   // source column
  int           buffer_pos;      // byte offset in the original SQL string
  char         *sql_user_text;   // pointer into original SQL (shared)

  /* --- list / tree links --- */
  PT_NODE      *next;            // next sibling in a NULL-terminated linked list
  PT_NODE      *or_next;         // next clause in a DNF predicate list
  PT_NODE      *next_row;        // row link inside PT_NODE_LIST

  /* --- type system --- */
  PT_TYPE_ENUM  type_enum;       // SQL type after type inference (PT_TYPE_NONE until resolved)
  PT_NODE      *data_type;       // PT_DATA_TYPE node for complex/parameterized types
  TP_DOMAIN    *expected_domain; // expected domain for input host-variable markers

  /* --- cross-references --- */
  UINTPTR       spec_ident;      // entity-spec equivalence class id (used by optimizer)
  XASL_ID      *xasl_id;         // filled in after XASL generation

  /* --- printing / alias --- */
  const char   *alias_print;     // column alias text (cached printed form)
  PARSER_VARCHAR *expr_before_const_folding; // pre-folding text for EXPLAIN

  /* --- miscellaneous --- */
  void         *etc;             // application scratch pointer (opaque)
  CACHE_TIME    cache_time;
  int           sub_host_var_count;
  int          *sub_host_var_index;

  /* --- packed flags --- */
  struct {
    unsigned recompile:1;
    unsigned cannot_prepare:1;
    unsigned partition_pruned:1;
    unsigned si_datetime:1;
    unsigned is_hidden_column:1;
    unsigned with_rollup:1;
    unsigned do_not_fold:1;
    unsigned is_value_query:1;
    unsigned is_system_generated_stmt:1;
    // … ~25 more single-bit flags
  } flag;

  /* --- per-node-type payload --- */
  PT_STATEMENT_INFO info;        // large anonymous union — access via info.<member>
};
```

## Node type enum (`PT_NODE_TYPE`)

~80 variants. The first block mirrors `CUBRID_STMT_*` values (so statement nodes double as API statement codes):

| Range | Examples |
|-------|---------|
| Statement nodes | `PT_SELECT`, `PT_INSERT`, `PT_UPDATE`, `PT_DELETE`, `PT_MERGE` |
| DDL nodes | `PT_CREATE_ENTITY`, `PT_ALTER`, `PT_DROP`, `PT_CREATE_INDEX`, `PT_TRUNCATE` |
| DCL/TCL nodes | `PT_GRANT`, `PT_REVOKE`, `PT_COMMIT_WORK`, `PT_ROLLBACK_WORK`, `PT_SAVEPOINT` |
| Set ops | `PT_UNION`, `PT_DIFFERENCE`, `PT_INTERSECTION` (assigned above `CUBRID_MAX_STMT_TYPE`) |
| Expression helpers | `PT_EXPR`, `PT_NAME`, `PT_VALUE`, `PT_FUNCTION`, `PT_HOST_VAR`, `PT_DOT_`, `PT_SPEC` |
| Type/constraint | `PT_DATA_TYPE`, `PT_ATTR_DEF`, `PT_CONSTRAINT`, `PT_SORT_SPEC` |
| Misc | `PT_CTE`, `PT_WITH_CLAUSE`, `PT_JSON_TABLE`, `PT_JSON_TABLE_COLUMN`, `PT_SHOWSTMT`, `PT_VACUUM`, `PT_DBLINK_TABLE` |

> [!key-insight] Function table indexed by node_type
> `parse_tree_cl.c` has static function tables for `parser_new_node`, `parser_init_node`, and `parser_print_tree` indexed by `PT_NODE_TYPE` ordinal. Adding a new node type requires adding entries to all three tables in exact order, or the process will crash.

## The info union — key members

### `PT_SELECT` — `info.query`

| Field | Type | Meaning |
|-------|------|---------|
| `from` | `PT_NODE *` | list of `PT_SPEC` |
| `where` | `PT_NODE *` | `PT_EXPR` |
| `select_list` | `PT_NODE *` | list of `PT_EXPR`/`PT_NAME`/`PT_FUNCTION` |
| `group_by` | `PT_NODE *` | list of `PT_SORT_SPEC` |
| `having` | `PT_NODE *` | `PT_EXPR` |
| `order_by` | `PT_NODE *` | list of `PT_SORT_SPEC` |
| `hint` | `PT_HINT_ENUM` | 64-bit bitmask |
| `all_distinct` | `PT_MISC_TYPE` | `PT_ALL` or `PT_DISTINCT` |
| `correlation_level` | `int` | 0 = non-correlated, >0 = correlated subquery depth |

### `PT_NAME` — `info.name`

| Field | Type | Meaning |
|-------|------|---------|
| `original` | `const char *` | identifier text as written |
| `resolved` | `const char *` | resolved class name (set by name_resolution) |
| `db_object` | `DB_OBJECT *` | resolved schema object pointer |
| `spec_id` | `UINTPTR` | back-reference to owning `PT_SPEC.id` |
| `meta_class` | `PT_MISC_TYPE` | `PT_NORMAL`/`PT_META_CLASS`/`PT_OID_ATTR`/`PT_PARAMETER`/… |

### `PT_EXPR` — `info.expr`

| Field | Type | Meaning |
|-------|------|---------|
| `op` | `PT_OP_TYPE` | operator: `PT_EQ`, `PT_AND`, `PT_PLUS`, `PT_CAST`, `PT_BETWEEN`, … |
| `arg1`, `arg2`, `arg3` | `PT_NODE *` | operands |
| `is_order_dependent` | bit | expression result depends on row order |
| `coll_modifier` | `int` | collation modifier (value + 1, 0 = none) |

### `PT_SPEC` — `info.spec`

Represents one table/view/subquery in the `FROM` clause.

| Field | Type | Meaning |
|-------|------|---------|
| `entity_name` | `PT_NODE *` | `PT_NAME` for real tables |
| `derived_table` | `PT_NODE *` | subquery node for derived tables |
| `cte_pointer` | `PT_NODE *` | pointer to `PT_CTE` node |
| `id` | `UINTPTR` | unique numeric id used as equivalence key |
| `join_type` | `PT_JOIN_TYPE` | `PT_JOIN_NONE`, `PT_JOIN_INNER`, `PT_JOIN_LEFT_OUTER`, … |
| `flag` | bitmask | `PT_SPEC_FLAG_KEY_INFO_SCAN`, `PT_SPEC_FLAG_RECORD_INFO_SCAN`, … |
| `flat_entity_list` | `PT_NODE *` | flattened class hierarchy list (set by name_resolution) |

## Type system

`PT_TYPE_ENUM` starts at 1000 to avoid clashing with `PT_NODE_TYPE`:

```
PT_TYPE_NONE  (1000)   — not yet resolved
PT_TYPE_INTEGER, PT_TYPE_BIGINT, PT_TYPE_SMALLINT
PT_TYPE_FLOAT, PT_TYPE_DOUBLE, PT_TYPE_NUMERIC, PT_TYPE_MONETARY
PT_TYPE_CHAR, PT_TYPE_VARCHAR, PT_TYPE_BIT, PT_TYPE_VARBIT
PT_TYPE_DATE, PT_TYPE_TIME, PT_TYPE_TIMESTAMP, PT_TYPE_DATETIME
PT_TYPE_TIMESTAMPTZ, PT_TYPE_TIMESTAMPLTZ, PT_TYPE_DATETIMETZ, PT_TYPE_DATETIMELTZ
PT_TYPE_JSON
PT_TYPE_OBJECT, PT_TYPE_SET, PT_TYPE_MULTISET, PT_TYPE_SEQUENCE
PT_TYPE_BLOB, PT_TYPE_CLOB
PT_TYPE_ENUMERATION
PT_TYPE_MAYBE  — indeterminate (host vars before binding)
PT_TYPE_NULL, PT_TYPE_NA, PT_TYPE_STAR
```

Type classification macros in `parse_tree.h`:

```c
PT_IS_NUMERIC_TYPE(t)       // INTEGER, BIGINT, SMALLINT, FLOAT, DOUBLE, MONETARY, NUMERIC, LOGICAL
PT_IS_STRING_TYPE(t)        // CHAR, VARCHAR, BIT, VARBIT
PT_IS_DATE_TIME_TYPE(t)     // DATE, TIME, TIMESTAMP, DATETIME + TZ variants
PT_IS_COLLECTION_TYPE(t)    // SET, MULTISET, SEQUENCE
PT_IS_LOB_TYPE(t)           // BLOB, CLOB
PT_HAS_COLLATION(t)         // CHAR, VARCHAR, ENUMERATION
PT_IS_PARAMETERIZED_TYPE(t) // NUMERIC, VARCHAR, CHAR, VARBIT, BIT, ENUMERATION
```

## Traversal API

```c
// walker callback signature
typedef PT_NODE *(*PT_NODE_WALK_FUNCTION)(
    PARSER_CONTEXT *parser,
    PT_NODE *node,
    void *arg,
    int *continue_walk);   // output: PT_STOP_WALK | PT_CONTINUE_WALK | PT_LEAF_WALK | PT_LIST_WALK

// traverse entire tree
parser_walk_tree(parser, root, pre_fn, pre_arg, post_fn, post_arg);

// traverse children only (not root)
parser_walk_leaves(parser, node, pre_fn, pre_arg, post_fn, post_arg);
```

Walk control values:

| Value | Effect |
|-------|--------|
| `PT_STOP_WALK` | Stop the entire walk |
| `PT_CONTINUE_WALK` | Visit children and siblings |
| `PT_LEAF_WALK` | Visit children but not siblings |
| `PT_LIST_WALK` | Visit `next` siblings but not children |

## Pointer utilities

```c
pt_point(parser, tree)          // shallow pointer wrapper (PT_NODE_POINTER)
CAST_POINTER_TO_NODE(p)         // dereference through PT_NODE_POINTER chain (macro)
parser_append_node(node, list)  // append to NULL-terminated list; returns new list head
```

## Typed accessor macros (debug-safe)

In debug builds these assert node_type; in release builds they are zero-overhead:

```c
PT_EXPR_OP(n)         // ((n)->info.expr.op)
PT_EXPR_ARG1(n)       // ((n)->info.expr.arg1)
PT_NAME_ORIGINAL(n)   // ((n)->info.name.original)
PT_NAME_DB_OBJECT(n)  // ((n)->info.name.db_object)
PT_SPEC_ID(n)         // ((n)->info.spec.id)
PT_SPEC_ENTITY_NAME(n)
```

## Hints stored on the node

`PT_HINT_ENUM` is a `UINT64` bitmask attached to query nodes (`info.query.hint`). It has 47 defined bits (as of v11.5) covering index hints, join method hints, cache hints, parallelism hints (`PT_HINT_PARALLEL`, `PT_HINT_NO_PARALLEL_HEAP_SCAN`, etc.), and CTE strategy hints (`PT_HINT_INLINE_CTE`, `PT_HINT_MATERIALIZE_CTE`).

## Invariants

- `node_type` is always set before any `info` field is accessed.
- `next` forms a NULL-terminated singly-linked list; never circular.
- `or_next` forms a DNF list used in predicate pushdown; independent of `next`.
- `type_enum == PT_TYPE_NONE` until `type_checking.c` has processed the node.
- `info.name.spec_id` must match `info.spec.id` of the owning `PT_SPEC` after name resolution completes, or xasl_generation will silently produce wrong join plans.

## Related

- Parent: [[components/parser|parser]]
- [[components/name-resolution|name-resolution]] — binds `PT_NAME.db_object` and `PT_SPEC.flat_entity_list`
- [[components/semantic-check|semantic-check]] — validates structure; may replace nodes
- [[components/xasl-generation|xasl-generation]] — converts `PT_NODE` tree to `XASL_NODE`
- [[Memory Management Conventions]]
