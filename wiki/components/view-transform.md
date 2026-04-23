---
type: component
parent_module: "[[components/parser|parser]]"
path: "src/parser/view_transform.c, src/parser/method_transform.c"
status: active
purpose: "View/vclass inlining, method call rewriting, and updatability analysis — runs between semantic_check and xasl_generation"
key_files:
  - "view_transform.c (~8000 lines — mq_translate and friends)"
  - "view_transform.h (public API)"
  - "method_transform.c (method call transformation)"
public_api:
  - "mq_translate(parser, node) → PT_NODE* — main entry: inline all views in statement"
  - "mq_updatable(parser, statement) → PT_UPDATABILITY — check if statement can be updated"
  - "mq_is_updatable(vclass_object) → bool"
  - "mq_is_updatable_strict(vclass_object) → bool"
  - "mq_rewrite_aggregate_as_derived(parser, agg_sel) → PT_NODE*"
  - "mq_rewrite_query_as_derived(parser, query) → PT_NODE*"
  - "mq_get_references / mq_set_references / mq_reset_ids"
  - "mq_evaluate_expression(parser, expr, value, object, spec_id)"
  - "mq_evaluate_check_option(parser, expr, object, view_class)"
  - "mq_copypush_sargable_terms(parser, statement, spec)"
  - "mq_bump_correlation_level(parser, node, increment, match)"
tags:
  - component
  - cubrid
  - parser
  - views
  - transform
related:
  - "[[components/parser|parser]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/semantic-check|semantic-check]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# View Transform (`view_transform.c`)

Runs after [[components/semantic-check|semantic_check]] and before [[components/xasl-generation|XASL generation]]. Its job is to replace every view reference in the parse tree with the view's defining query, adjust correlation levels, and check whether the resulting statement is updatable.

## Role in the pipeline

```
pt_semantic_check  (validate structure)
    │
    ▼
mq_translate       (view_transform.c)
    │  replaces PT_SPEC (view) → derived PT_SPEC (subquery)
    │  rewrites PT_NAME references to match new spec structure
    │  adjusts correlation_level throughout
    │  checks PT_UPDATABILITY
    ▼
xasl_generate_statement
```

## Updatability model

```c
enum pt_updatability {
  PT_NOT_UPDATABLE       = 0x0,  // non-updatable (contains aggregates, DISTINCT, etc.)
  PT_PARTIALLY_UPDATABLE = 0x1,  // vclass with joins but otherwise updatable
  PT_UPDATABLE           = 0x3   // fully updatable
};
```

`mq_updatable(parser, statement)` walks the parse tree and returns the weakest updatability of all referenced views. A view is NOT updatable if its query contains:
- `DISTINCT`
- `GROUP BY` / `HAVING`
- Aggregate functions
- Set operations (`UNION`, etc.)
- More than one non-updatable base table (without join being one-to-one)

`mq_is_updatable_strict` applies stricter rules: the vclass must have exactly one base class, no joins.

`PT_FETCH_AS` enum controls how the view spec is fetched during inlining:

```c
typedef enum PT_FETCH_AS {
  PT_NORMAL_SELECT,        // normal read
  PT_INVERTED_ASSIGNMENTS, // update via inverted view expression
  PT_PARTIAL_SELECT        // select only referenced columns
} PT_FETCH_AS;
```

## Inlining mechanism (`mq_translate`)

For each `PT_SPEC` that resolves to a view:

1. Retrieve the view's query spec from the schema (`sm_get_class_with_purpose`).
2. Make the view query a derived table: wrap it in a new `PT_SPEC` with `derived_table = <view query>`.
3. Call `mq_reset_ids` to re-assign fresh `spec_id` values, avoiding clashes with the outer query's ids.
4. Call `mq_set_references` to update all `PT_NAME.spec_id` references inside the inlined subquery to point at the new ids.
5. Call `mq_bump_correlation_level` to increment the `correlation_level` of any names that were correlated to outer scopes and are now one level deeper.
6. Recursively call `mq_translate` on the inlined subquery (views can reference other views).

`mq_make_derived_spec` is the helper that constructs the new `PT_SPEC` wrapper.

## Sargable term pushdown

`mq_copypush_sargable_terms` copies predicates from the outer query's WHERE clause into the inlined view subquery WHERE clause when they are sargable (can be pushed). This improves index usage inside the view.

## Expression evaluation for triggers/CHECK OPTION

`mq_evaluate_expression` can evaluate a `PT_EXPR` directly against an object using the schema `DB_OBJECT *`. This is used for:
- Evaluating `WITH CHECK OPTION` on view updates.
- Evaluating trigger conditions.
- `mq_evaluate_check_option` specifically verifies that an INSERT/UPDATE through a view satisfies the view's WHERE clause.

## Aggregate rewriting

`mq_rewrite_aggregate_as_derived` rewrites:

```sql
SELECT a, SUM(b) FROM t GROUP BY a
```

into a derived-table form when the aggregate is inside a subquery that is being inlined. This ensures the aggregate is evaluated at the correct scope level.

`mq_rewrite_query_as_derived` is the more general form that wraps any query as a derived table.

## Reference management

| Function | Purpose |
|----------|---------|
| `mq_get_references` | Collect all PT_NAME nodes referencing a given spec |
| `mq_get_references_helper` | Version that also collects referenced attributes |
| `mq_set_references` | Update PT_NAME nodes after spec ID change |
| `mq_reset_paths` | Reset path expressions after spec rewrite |
| `mq_reset_ids` | Reassign fresh spec IDs to avoid collisions |
| `mq_clear_ids` | Clear spec ID on nodes that no longer belong to a spec |
| `mq_reset_ids_in_statement` | Top-level ID reset for a full statement |
| `mq_reset_ids_in_methods` | ID reset specifically for method call nodes |

## Click counter handling

`mq_has_click_counter` is a walk callback that detects `INCR()`/`DECR()` click counter expressions in the tree, flagging them so the update path can special-case them. Click counters require server-side atomic increment and cannot go through the normal update machinery.

## Correlation level adjustment

```c
PT_NODE *mq_bump_correlation_level(
    PARSER_CONTEXT *parser, PT_NODE *node,
    int increment, int match);
```

When a subquery is inlined as a derived table, names correlated to outer queries are now one scope level further out. `mq_bump_correlation_level` increments `info.query.correlation_level` by `increment` for all subqueries whose correlation level equals `match`.

## Related

- Parent: [[components/parser|parser]]
- [[components/parse-tree|parse-tree]] — PT_SPEC, PT_NAME structures modified here
- [[components/semantic-check|semantic-check]] — previous pass
- [[components/xasl-generation|xasl-generation]] — next pass; sees inlined tree
- [[Query Processing Pipeline]]
