---
type: component
parent_module: "[[components/optimizer-rewriter]]"
path: "src/optimizer/rewriter/query_rewrite_subquery.c"
status: active
purpose: "Subquery rewrites: turn uncorrelated equality / quantified-comparison subqueries into joins with derived tables; isolate hidden-column subqueries; force LIMIT 1 on EXISTS subqueries"
key_files:
  - "query_rewrite_subquery.c (497 LOC, 3 public functions)"
public_api:
  - "qo_rewrite_subqueries(parser, node, arg, continue_walk) → PT_NODE * — pre-walker that converts uncorrelated subqueries to joins"
  - "qo_rewrite_hidden_col_as_derived(parser, node, parent_node) → PT_NODE * — wrap an ORDER-BY-bearing subquery as a derived table when it carries hidden columns; remove unnecessary ORDER BY"
  - "qo_add_limit_clause(parser, node) → void — append LIMIT 1 to a subquery that has none and no INST_NUM/ORDERBY_NUM/GROUPBY_NUM"
tags:
  - component
  - cubrid
  - optimizer
  - rewriter
  - subquery
related:
  - "[[components/optimizer-rewriter]]"
  - "[[components/parser]]"
  - "[[components/parse-tree]]"
created: 2026-04-25
updated: 2026-04-25
---

# `query_rewrite_subquery.c` — Subquery Rewrites

Three functions, ~500 LOC. Owns the structural transformations that change a subquery's relational position in the tree (subquery → join, subquery → derived table, EXISTS subquery → LIMIT 1 subquery). Predicate-level subquery handling lives in [[components/optimizer-rewriter-term]] and [[components/optimizer-rewriter-select]].

## `qo_rewrite_subqueries` — uncorrelated subquery to join (`:39-318`)

```c
PT_NODE *qo_rewrite_subqueries (PARSER_CONTEXT *parser, PT_NODE *node, void *arg, int *continue_walk);
```

Run as a `parser_walk_tree` pre-callback. Sets `*continue_walk = PT_LIST_WALK` at the end (so it descends into children but does not re-process the modified parent).

Targets these operator shapes in the WHERE clause:

| Operator | Form |
|---|---|
| `PT_EQ` | `attr = (SELECT col FROM ...)` (single-col only) and `(c1, c2) = (SELECT a, b FROM ...)` (multi-col tuple) |
| `PT_IS_IN` | `(c1, c2) IN (SELECT a, b FROM ...)` (collection types only) |
| `PT_EQ_SOME` | `attr = SOME (SELECT col FROM ...)` |
| `PT_GT_SOME`, `PT_GE_SOME`, `PT_LT_SOME`, `PT_LE_SOME` | `attr {>,>=,<,<=} SOME (SELECT col FROM ...)` |

**Single-column `PT_EQ` is rejected** at `:100-104` — `col1 = (SELECT col1 FROM t)` is intentionally NOT rewritten to a join with derived table. Comment: "one column subquery is not rewrited to join with derived table." (likely because the post-rewrite plan would force semantics other than "fail if subquery returns >1 row".)

### Eligibility check

Both sides must satisfy:
- `tp_valid_indextype` — the type must be indexable (no LOB/JSON/SET).
- The LHS column-by-column must be `pt_is_attr` or `pt_is_function_index_expression`.
- The subquery must have `correlation_level == 0` (uncorrelated).
- The subquery must not have analytic functions (`pt_has_analytic`).
- For multi-column form, both arrays must zip cleanly.

If any column fails, the whole CNF term is skipped — no partial rewrite.

### `PT_EQ` / `PT_IS_IN` / `PT_EQ_SOME` rewrite (`:154-221`)

1. If RHS is a `PT_FUNCTION` collection or `PT_VALUE` collection, `pt_select_list_to_one_col` collapses the subquery select-list.
2. `mq_make_derived_spec` creates a new derived spec and appends it to FROM.
3. The original `PT_EQ`-class CNF term is rewritten to `attr = derived_attr`.
4. For multi-column tuples, additional `PT_EQ` CNF terms are spliced into the WHERE list (one per column pair).
5. Recurses into the new derived table's subquery via `parser_walk_tree(qo_rewrite_subqueries)` to handle nested cases.

### Quantified comparison rewrite (`:223-306`)

For `attr {>,>=,<,<=} SOME (subquery)`, the subquery's select list is replaced with `MIN(col)` (for `>`/`>=`) or `MAX(col)` (for `<`/`<=`):

| Operator | Aggregate | Resulting comparison |
|---|---|---|
| `PT_GT_SOME` | `MIN(col)` | `attr > MIN(col)` |
| `PT_GE_SOME` | `MIN(col)` | `attr >= MIN(col)` |
| `PT_LT_SOME` | `MAX(col)` | `attr < MAX(col)` |
| `PT_LE_SOME` | `MAX(col)` | `attr <= MAX(col)` |

If the subquery is composite (`UNION`/`INTERSECTION`/`DIFFERENCE`) or already aggregated or has `orderby_for`, it is first wrapped via `mq_rewrite_query_as_derived` — composite quantified comparisons can't be rewritten in-place.

The aggregate is built as a `PT_FUNCTION` node with `function_type = PT_MIN` or `PT_MAX`, `all_or_distinct = PT_ALL`. The select list is moved into `arg_list` of the new function. `PT_SELECT_INFO_HAS_AGG` flag is set on the subquery so downstream plan generation treats it as an aggregating SELECT.

### What about `_ALL` quantifiers?

`PT_GT_ALL`, `PT_GE_ALL`, `PT_LT_ALL`, `PT_LE_ALL` are NOT in the trigger list at `:73-76`. `attr > ALL (subq)` is left alone here and handled (if at all) elsewhere in the rewriter or planner.

## `qo_rewrite_hidden_col_as_derived` — ORDER-BY hidden column isolation (`:330-464`)

```c
PT_NODE *qo_rewrite_hidden_col_as_derived (PARSER_CONTEXT *parser, PT_NODE *node, PT_NODE *parent_node);
```

Two purposes, both about ORDER BY interaction with hidden columns (columns added to the select list to support `ORDER BY` on expressions not in the user's select list):

### Purpose A: Remove unnecessary ORDER BY

When a SELECT has `ORDER BY` but is in a context where ordering is irrelevant (e.g. `EXISTS (SELECT ... ORDER BY x)` — order doesn't affect the EXISTS truth value), the ORDER BY is freed. Conditions:

- `parent_node` is NULL (top-level subquery context that doesn't preserve order) OR `parent_node` is a `PT_FUNCTION` whose `function_type` is `F_TABLE_SEQUENCE` (the only collection type that DOES preserve order).
- No `orderby_for` (`ORDER BY ... FOR <num>`).
- No `PT_ORDERBY_NUM` pseudo-column in the select list.
- Not a `CONNECT BY` query.

If all conditions hold, `parser_free_tree(order_by)` releases the clause and the trailing hidden column is freed too.

### Purpose B: Wrap as derived if hidden columns must be preserved

When ORDER BY can't be removed but the subquery is a `PT_IS_SUBQUERY`, the entire subquery is wrapped via `mq_rewrite_query_as_derived` so the hidden-column structure is encapsulated and doesn't leak into the outer query's projection.

Special case: if **all** select-list nodes are hidden columns, the wrap is **skipped** to avoid producing a derived table with a NULL select list. Comment cites the test case `set @a = 1; SELECT (SELECT @a := @a + 1 FROM db_root ORDER BY @a + 1)` which would otherwise crash.

For `PT_UNION`/`PT_DIFFERENCE`/`PT_INTERSECTION`, both arms are recursively processed.

## `qo_add_limit_clause` — force `LIMIT 1` on EXISTS subqueries (`:472-497`)

```c
void qo_add_limit_clause (PARSER_CONTEXT *parser, PT_NODE *node);
```

Called from the orchestrator's `PT_EXISTS` arm (`query_rewrite.c:246-251`). Adds `LIMIT 1` to the subquery so the plan can short-circuit on the first matching row.

Bail-out if any of:
- `info.query.limit` is already set.
- WHERE contains `INST_NUM()` (`pt_check_instnum_pre/post`).
- `orderby_for` contains `ORDERBY_NUM()` (`pt_check_orderbynum_pre/post`).
- HAVING contains `GROUPBY_NUM()` (`pt_check_groupbynum_pre/post`).

Otherwise, builds a `PT_VALUE` node with `type_enum=PT_TYPE_INTEGER`, `data_value.i=1`, and assigns it to `info.query.limit`. Sets `flag.rewrite_limit = 1`.

> [!key-insight] EXISTS optimization is grammar-rewriting, not planner trick
> The "exists short-circuits on first match" optimization is implemented at the parse-tree level by injecting a `LIMIT 1`. The planner sees no special EXISTS-aware semantics — it just plans a subquery with LIMIT 1, which any sane plan executes by stopping at the first row.

## Smells / observations

- Single-column `PT_EQ` skip at `:100-104` is asymmetric with multi-column tuple support. The comment doesn't justify the choice; downstream handling presumably catches single-column equality elsewhere.
- `qo_add_limit_clause` builds a `PT_VALUE` integer 1 without setting `info.value.location`. If a later pass uses location for outer-join recovery, this LIMIT slot is location-0 (which is the "WHERE" location). Probably fine because LIMIT is not outer-join-sensitive, but worth noting.
- `qo_rewrite_hidden_col_as_derived` is defensive about the empty-derived-select-list case — has a regression-test reference baked into the comment (the `@a := @a + 1` example).
- `mq_rewrite_query_as_derived` is the workhorse for "make this query a derived spec"; lives in `view_transform.c` (parser-side multi-query helper).

## Cross-references

- Calls: `mq_make_derived_spec`, `mq_rewrite_query_as_derived`, `pt_get_select_list`, `pt_select_list_to_one_col`, `pt_is_attr`, `pt_is_function_index_expression`, `pt_has_analytic`, `tp_valid_indextype`, `pt_type_enum_to_db`, `pt_check_instnum_pre/post`, `pt_check_orderbynum_pre/post`, `pt_check_groupbynum_pre/post`, `pt_length_of_select_list`, `parser_new_node`, `parser_copy_tree`, `parser_free_tree`, `parser_append_node`.
- Macros: `PT_NODE_MOVE_NUMBER_OUTERLINK`, `PT_SELECT_INFO_SET_FLAG`, `PT_IS_NULL_NODE`, `PT_IS_COLLECTION_TYPE`, `PT_IS_FUNCTION`, `PT_IS_CONST`, `PT_IS_SELECT`.

## Related

- Parent: [[components/optimizer-rewriter]]
- Sibling: [[components/optimizer-rewriter-term]] — predicate-level subquery normalization that runs after this pass.
