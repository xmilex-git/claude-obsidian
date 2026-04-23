---
type: component
parent_module: "[[components/parser|parser]]"
path: "src/parser/xasl_generation.c, src/parser/xasl_generation.h"
status: active
purpose: "Convert the validated PT_NODE parse tree into an XASL_NODE execution plan; the last client-side step before serialisation to the server"
key_files:
  - "xasl_generation.c (~25 000 lines — main implementation)"
  - "xasl_generation.h (SYMBOL_INFO, TABLE_INFO, AGGREGATE_INFO, ANALYTIC_INFO, COMPATIBLE_INFO)"
  - "xasl.h (XASL_NODE definition — src/xasl/)"
  - "regu_var.hpp (REGU_VARIABLE — runtime register allocation)"
  - "optimizer.h (QO_PLAN consumed here)"
public_api:
  - "xasl_generate_statement(parser, statement) → XASL_NODE*"
  - "pt_to_regu_variable(parser, node, unbox) → REGU_VARIABLE*"
  - "pt_make_pred_expr_pred(parser, node, sc_info, arg) → PRED_EXPR*"
  - "pt_make_dblink_access_spec(access, pred, pred_list, attr_list, url, user, …)"
tags:
  - component
  - cubrid
  - parser
  - xasl
  - query
related:
  - "[[components/parser|parser]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/xasl|xasl]]"
  - "[[components/optimizer|optimizer]]"
  - "[[components/semantic-check|semantic-check]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# XASL Generation (`xasl_generation.c`)

The final client-side compilation step. Translates a fully-resolved, type-checked `PT_NODE` tree into an `XASL_NODE` tree that can be serialised and shipped to the server for execution.

## Role in the pipeline

```
PT_NODE (type-checked, views inlined)
    │
    │  xasl_generate_statement()
    ▼
XASL_NODE tree
    │
    │  xasl_to_stream()   (client-side serialisation)
    │  ──── TCP ────────
    │  stream_to_xasl()   (server-side deserialisation)
    ▼
query_executor.c  →  scan_manager.c  →  …
```

## Key data structures (xasl_generation.h)

### SYMBOL_INFO

A stack of scopes maintained during XASL generation. Each nested `SELECT` pushes a new frame:

```c
struct symbol_info {
  SYMBOL_INFO   *stack;            // enclosing scope
  TABLE_INFO    *table_info;       // list of visible tables + their value lists
  PT_NODE       *current_class;    // class being resolved
  HEAP_CACHE_ATTRINFO *cache_attrinfo;
  PT_NODE       *current_listfile; // list file for derived tables
  VAL_LIST      *listfile_value_list;
  UNBOX          listfile_unbox;   // UNBOX_AS_VALUE or UNBOX_AS_TABLE
  int            listfile_attr_offset;
  PT_NODE       *query_node;       // the PT_SELECT being translated
  DB_VALUE     **reserved_values;  // reserved attribute db_values
};
```

### TABLE_INFO

One entry per `PT_SPEC` visible in the current query scope:

```c
struct table_info {
  struct table_info *next;
  PT_NODE   *class_spec;       // SHARED pointer to the PT_SPEC parse node
  const char *exposed;         // alias name (shared pointer)
  UINTPTR    spec_id;          // == PT_SPEC.id
  PT_NODE   *attribute_list;   // PT_NAME list of referenced columns
  VAL_LIST  *value_list;       // parallel DB_VALUE list for column evaluation
  int        is_fetch;
};
```

`attribute_list` and `value_list` are built during XASL generation; `REGU_VARIABLE` nodes reference into `value_list` slots rather than directly to the underlying storage.

### AGGREGATE_INFO / ANALYTIC_INFO

Accumulate aggregate (`GROUP BY`) and analytic (window function) nodes respectively as the SELECT list is translated. Both hold head-list pointers and output-pointer lists that are attached to the final `XASL_NODE`.

### UNBOX enum

```c
typedef enum { UNBOX_AS_VALUE, UNBOX_AS_TABLE } UNBOX;
```

Controls whether a subquery result is treated as a scalar or as a derived table. Passed to `pt_to_regu_variable` when translating subquery nodes.

## Translation process

### `xasl_generate_statement`

Top-level driver. Dispatches on `statement->node_type`:

- `PT_SELECT` → `pt_to_buildlist_proc` (or `pt_to_buildvalue_proc` for VALUES queries)
- `PT_INSERT` → `pt_to_insert_xasl`
- `PT_UPDATE` → `pt_to_update_xasl`
- `PT_DELETE` → `pt_to_delete_xasl`
- `PT_MERGE` → `pt_to_merge_xasl`
- `PT_UNION/DIFFERENCE/INTERSECTION` → `pt_to_union_proc`

### `pt_to_regu_variable`

Translates a single `PT_NODE` expression into a `REGU_VARIABLE` — the runtime "register" used by the XASL interpreter:

```c
REGU_VARIABLE *pt_to_regu_variable(
    PARSER_CONTEXT *parser,
    PT_NODE *node,
    UNBOX unbox);
```

Dispatch on `node->node_type`:
- `PT_NAME` → `REGU_TYPE_ATTR_ID` (slot in `VAL_LIST`)
- `PT_VALUE` → `REGU_TYPE_DBVAL` (immediate value)
- `PT_HOST_VAR` → `REGU_TYPE_HVAR` (host variable index)
- `PT_EXPR` → recursive call for args, then `REGU_TYPE_OPER`
- `PT_FUNCTION` → `REGU_TYPE_FUNC`
- `PT_SELECT` (subquery) → `REGU_TYPE_SUBQUERY`

### Predicate translation

`PT_EXPR` predicate nodes are translated to `PRED_EXPR` (in `cubxasl::pred_expr`) via `pt_to_pred_expr`. Boolean AND/OR trees are recursively translated. `PT_PRED_ARG_INSTNUM_CONTINUE` / `PT_PRED_ARG_GRBYNUM_CONTINUE` / `PT_PRED_ARG_ORDBYNUM_CONTINUE` flags indicate that the predicate includes special counters (`ROWNUM`, `GROUPBY_NUM`, `ORDERBY_NUM`) that require deferred evaluation.

### Optimizer plan consumption

Before emitting the final `XASL_NODE`, the optimizer (`query_planner.c`) is called to produce a `QO_PLAN *`. The XASL generator reads the plan to determine:
- Access method per table (`ACCESS_SPEC_TYPE`: heap scan, index scan, index skip-scan, BTREE node scan, DBLink scan)
- Join method per join node (nested-loop, index, merge, hash)
- Partition pruning flags

`query_Plan_dump_filename` / `query_Plan_dump_fp` are module-level globals that redirect plan dump output when `SET TRACE ON` is in effect.

## Hash attribute analysis

A `HASHABLE` struct and a `HASH_ATTR` enum (`UNHASHABLE`, `PROBE`, `BUILD`, `CONSTANT`) are used during hash-join XASL generation to classify which side of each join attribute can be hashed. This analysis is internal to `xasl_generation.c`.

## Subquery cache integration

`subquery_cache.h` is included; `xasl_generation.c` annotates uncorrelated subqueries so the server-side subquery cache can avoid re-execution.

## DBLINK support

`pt_make_dblink_access_spec` constructs a special `ACCESS_SPEC_TYPE` for remote (DBLINK) table references, embedding the connection URL, credentials, and remote column list. This is a recent addition to the file.

## Analytic function optimisation

```c
#define ANALYTIC_OPT_MAX_SORT_LIST_COLUMNS  32
#define ANALYTIC_OPT_MAX_FUNCTIONS          32
```

The generator detects when multiple analytic functions share a PARTITION BY / ORDER BY key and merges their sort lists to avoid redundant sorts. Up to 32 sort columns and 32 functions are optimised in one pass.

## Invariants

- This pass is read-only with respect to `PT_NODE`; it builds a new `XASL_NODE` tree without modifying the parse tree (enabling re-use for `PREPARE` / plan cache).
- `SYMBOL_INFO` is pushed/popped in strict LIFO order; leaking a frame causes wrong attribute resolution in outer queries.
- Every `VAL_LIST` slot must have a corresponding `DB_VALUE` allocated in the `XASL_NODE`'s `dbval_ptr` array (maintained by `dbval_cnt` in `PARSER_CONTEXT`).

## Related

- Parent: [[components/parser|parser]]
- [[components/xasl|xasl]] — XASL_NODE type definitions
- [[components/optimizer|optimizer]] — QO_PLAN consumed here
- [[components/parse-tree|parse-tree]] — source PT_NODE tree
- [[components/semantic-check|semantic-check]] — previous pass; sets type_enum used here
- [[Query Processing Pipeline]]
