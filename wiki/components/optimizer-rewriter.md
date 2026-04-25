---
type: component
parent_module: "[[modules/src|src]]"
path: "src/optimizer/rewriter/"
status: active
purpose: "Orchestrator for the client-side query-rewrite pass: walks the parse tree, dispatches statement-specific pre-rewrites, applies CNF normalization, runs the rewrite stages, and finishes with auto-parameterization and post-walk fix-ups"
key_files:
  - "query_rewrite.c (739 LOC) — entry point mq_rewrite + qo_rewrite_queries dispatcher"
  - "query_rewrite.h (172 LOC) — internal-only header (MUST NOT be included outside this folder); shared structs and macros"
public_api:
  - "mq_rewrite(parser, statement) → PT_NODE * — the only externally-called entry point; wraps parser_walk_tree(qo_rewrite_queries, qo_rewrite_queries_post)"
tags:
  - component
  - cubrid
  - optimizer
  - rewriter
  - query
related:
  - "[[components/optimizer]]"
  - "[[components/optimizer-rewriter-select]]"
  - "[[components/optimizer-rewriter-term]]"
  - "[[components/optimizer-rewriter-subquery]]"
  - "[[components/optimizer-rewriter-set]]"
  - "[[components/optimizer-rewriter-auto-parameterize]]"
  - "[[components/optimizer-rewriter-unused-function]]"
  - "[[components/parser]]"
  - "[[components/parse-tree]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-25
updated: 2026-04-25
---

# `src/optimizer/rewriter/` — Query-rewrite Subsystem

The rewriter rewrites the parse-tree **before** the cost-based planner sees it. Eight files, ~10K LOC total, all client-side (`#if !defined(SERVER_MODE)`). Operates on `PT_NODE` trees produced by [[components/parser]] after name resolution and semantic check; output feeds [[components/optimizer]].

## Files in this directory

| File | LOC | Page |
|---|---:|---|
| `query_rewrite.c` | 739 | this page |
| `query_rewrite.h` | 172 | this page |
| `query_rewrite_select.c` | 3,795 | [[components/optimizer-rewriter-select]] |
| `query_rewrite_term.c` | 4,294 | [[components/optimizer-rewriter-term]] |
| `query_rewrite_subquery.c` | 497 | [[components/optimizer-rewriter-subquery]] |
| `query_rewrite_auto_parameterize.c` | 361 | [[components/optimizer-rewriter-auto-parameterize]] |
| `query_rewrite_set.c` | 153 | [[components/optimizer-rewriter-set]] |
| `query_rewrite_unused_function.c` | 201 | [[components/optimizer-rewriter-unused-function]] |

## Public entry point

```c
PT_NODE *mq_rewrite (PARSER_CONTEXT *parser, PT_NODE *statement);
```

Defined at `query_rewrite.c:735-739`. Single line of meaningful code:

```c
return parser_walk_tree (parser, statement, qo_rewrite_queries, NULL, qo_rewrite_queries_post, NULL);
```

The `mq_` prefix is historical — the function lives in the optimizer's rewriter directory but uses the multi-query-rewrite naming convention shared with `view_transform.c` and `parser/`. Callers expect the rewriter to be a single black-box pass.

## Header contract

`query_rewrite.h:20` carries the comment:

> don't include this except files in this folder

The header declares the cross-file rewriter API consumed by `query_rewrite_select.c`, `query_rewrite_term.c`, `query_rewrite_subquery.c`, `query_rewrite_set.c`, and `query_rewrite_auto_parameterize.c`. External consumers must call `mq_rewrite` only; nothing else is exported.

### Shared types (header)

| Type | Purpose |
|---|---|
| `SPEC_ID_INFO {id, appears, nullable}` | Records whether a `spec_id` appears in a sub-tree and whether the spec is on the nullable side of an outer join. |
| `SPEC_CNT_INFO {spec, my_spec_cnt, other_spec_cnt, my_spec_node}` | Counts references to a specific spec vs. all other specs — used by reduction passes that need to know "is this term local to one spec?" |
| `TO_DOT_INFO {old_spec, new_spec}` | Used when rewriting a spec ID across a `PT_DOT_` traversal. |
| `PT_NAME_SPEC_INFO {c_name, c_name_num, query_serial_num, s_point_list}` | Maps an attribute that will be reduced to a constant to its joining peer specs. |
| `RESET_LOCATION_INFO {start_spec, start, end, found_outerjoin}` | Tracks contiguous outer-join location ranges when relocating ON-conditions. |
| `QO_REDUCE_REFERENCE_INFO {pk_spec, pk_mop, pk_cons, fk_spec, fk_cons, exclude_pk_spec_point_list, exclude_fk_spec_point_list, join_pred_point_list, parent_pred_point_list, append_not_null_pred_list}` | State for the FK→PK join elimination optimization (eliminate parent table joins when the foreign key is non-null and equality-joined to its referenced PK). |

### Comparison-result enums

```c
enum comp_dbvalue_with_optye_result {
  CompResultLess       = -2,  /* less than */
  CompResultLessAdj    = -1,  /* less than and adjacent to */
  CompResultEqual      =  0,
  CompResultGreaterAdj =  1,  /* greater than and adjacent to */
  CompResultGreater    =  2,
  CompResultError      =  3
};

enum dnf_merge_range_result {
  DNF_RANGE_VALID         = 0,
  DNF_RANGE_ALWAYS_FALSE  = 1,
  DNF_RANGE_ALWAYS_TRUE   = 2
};
```

The "adjacent to" values mark the case where the boundary values differ by exactly one in a discrete domain (used by range merging in [[components/optimizer-rewriter-term]]).

### Macros

`QO_CHECK_AND_REDUCE_EQUALITY_TERMS(parser, node, where)` — idempotency guard around `qo_reduce_equality_terms`. Sets `node->flag.done_reduce_equality_terms = true` so a second call on the same subtree no-ops. Used because the equality-reduction is invoked from multiple WHERE-shaped slots (HAVING, START WITH, CONNECT BY, after-CB filter, MERGE update/insert/delete) and each slot must reduce at most once.

`PROCESS_IF_EXISTS(parser, condition, func)` — null-pointer-safe wrapper for the auto-parameterize pass.

## `qo_rewrite_queries` — per-statement dispatcher

At `query_rewrite.c:43-574`. Called as the **pre** function of `parser_walk_tree`. Invoked once per query node (top-level statement and every nested subquery).

Flow has three phases per statement:

### Phase 1: Pre-rewrite (statement-shape-specific)

A `switch (node->node_type)` selects which pointers to bind for the rewrite slots (`wherep`, `havingp`, `startwithp`, `connectbyp`, `aftercbfilterp`, `merge_upd_wherep`, `merge_ins_wherep`, `merge_del_wherep`, `orderby_for_p`, `show_argp`).

| Statement | Pre-rewrite actions |
|---|---|
| `PT_SELECT` | (a) For hierarchical queries (`CONNECT BY`), `pt_split_join_preds` separates join predicates from the after-CONNECT-BY filter so the two parts get optimized independently. If the query has no joins, the optimizer rewrites it to use the START WITH list as the WHERE clause and sets `single_table_opt=1`. (b) `qo_move_on_of_explicit_join_to_where` lifts every explicit-join `ON`-condition into the WHERE clause for unified rewriting (recovered post-walk by `qo_rewrite_queries_post` using `location` markers planted in `pt_bind_names`). (c) `qo_rewrite_index_hints` resolves hint name references. |
| `PT_UPDATE`, `PT_DELETE` | Same `qo_move_on_of_explicit_join_to_where` + `qo_rewrite_index_hints` treatment. |
| `PT_INSERT` | Only descends into a `SELECT`-shaped INSERT source (via `pt_get_subquery_of_insert_select`). VALUES INSERTs return early. |
| `PT_MERGE` | Binds 4 search-condition slots (top-level WHERE + update/insert/delete WHEREs). |
| `PT_UNION`/`PT_DIFFERENCE`/`PT_INTERSECTION` | Recursively rewrites the two arms. If a `LIMIT` exists without `ORDER BY`, the union is rewritten as a derived table wrapped in `SELECT * FROM (union) WHERE INST_NUM() <= limit` so the limit can push down. |
| `PT_EXPR` (subquery operator) | Wraps the operand subquery as a derived table for hidden-column-safe execution; for `PT_EXISTS` adds `LIMIT 1` to the subquery via `qo_add_limit_clause`. |
| `PT_FUNCTION` (table-set/multiset/sequence) | Same hidden-column-as-derived treatment for the argument subquery. |
| Other types | Return immediately — no WHERE to rewrite. |

### Phase 2: Optimization (gated on `OPTIMIZATION_ENABLED`)

`qo_get_optimization_param(QO_PARAM_LEVEL)` is checked at `query_rewrite.c:281-283`. When enabled:

1. `qo_rewrite_subqueries` (uncorrelated-subquery → join-with-derived) — only on SELECT.
2. **CNF conversion** of every WHERE-shaped slot via `pt_cnf` (defined in parser layer). Idempotent.
3. **HAVING → WHERE migration** for terms that don't reference aggregates or pseudo-columns and aren't blocked by `WITH ROLLUP`. Implementation walks each CNF node, runs `pt_has_non_groupby_column_node` + `pt_find_aggregate_functions_pre/post` + `pt_is_pseudocolumn_node`, and if all three say "movable" detaches the node from HAVING and appends to WHERE.
4. **Equality-term reduction** — `qo_reduce_equality_terms` for HAVING/MERGE/AFTER-CB-FILTER slots; for SELECT WHERE the **post** variant `qo_reduce_equality_terms_post` runs as `parser_walk_tree`'s post function so subqueries get reduced first (so e.g. `(select col1 from t where col1=1) a` produces `(select 1 from t where col1=1) a` and the outer `a.col1 = b.col1` becomes `1 = b.col1`). START WITH and CONNECT BY are deliberately NOT equality-reduced — see comment at `:454-458`: an equality `A = 5` in CONNECT BY would replace every later `A` reference with the literal `5`, breaking `ORDER BY A` in hierarchical results.
5. `qo_rewrite_terms` — applied to all 8 slot variants (WHERE/HAVING/START WITH/CONNECT BY/after-CB filter/3× MERGE).
6. `qo_rewrite_select_queries` — SELECT-specific transformations.

### Phase 3: Auto-parameterize (last)

Constants → host-variable input markers, gated on **all** of:

```c
!prm_get_bool_value (PRM_ID_HOSTVAR_LATE_BINDING)
&& prm_get_integer_value (PRM_ID_XASL_CACHE_MAX_ENTRIES) > 0
&& node->flag.cannot_prepare == 0
&& parser->flag.is_parsing_static_sql == 0
&& parser->flag.is_skip_auto_parameterize == 0
```

> [!key-insight] Auto-parameterize must be the LAST step
> Comment at `:494`: "auto-parameterization is safe when it is done as the last step of rewrite optimization". Earlier rewrites (constant folding, equality reduction) need to see literal values; once they're replaced with `?` markers the rewrites can no longer reason about specific values.

Even when the predicate-level auto-parameterize is skipped, two narrow paths still run unconditionally:
- WHERE clauses with `flag.force_auto_parameterize = true` (set by upstream rewrites that demand parameterization).
- LIMIT and KEYLIMIT clauses — `qo_auto_parameterize_limit_clause` + `qo_auto_parameterize_keylimit_clause`.
- `SHOW`-statement scalar arguments — `pt_rewrite_to_auto_param` per arg.

For `PT_UPDATE`, the assignment list is also auto-parameterized when the gate is on (`:526-529`).

## `qo_rewrite_queries_post` — post-walk fix-up

At `query_rewrite.c:585-725`. The pre-pass merged outer-join `ON`-conditions into WHERE for uniform rewriting; this post-pass walks the resulting WHERE list and:

1. For each term with `info.expr.location > 0`, locates the matching FROM-spec by `info.spec.location`.
2. If the spec is `PT_JOIN_LEFT_OUTER` / `PT_JOIN_RIGHT_OUTER` / `PT_JOIN_INNER`, the term is **moved back** to the spec's `on_cond` list (preserving outer-join semantics).
3. If the join was already converted to inner (by some earlier rewrite step), the location is cleared and the term stays in WHERE — except `PT_EXPR_INFO_COPYPUSH`-flagged terms (predicate copy-push artefacts) which are freed.

Failure mode: a `location > 0` with no matching spec produces `PT_ERRORf "check outer join syntax at '<term>'"` and the term stays in WHERE.

> [!warning] Outer-join correctness boundary
> Any rewrite that runs between the pre-pass and post-pass MUST preserve `info.expr.location` on terms that originated from `ON`-conditions. Losing the location turns an outer-join filter into an inner-join filter — silent semantic change.

## Slot binding table

Per statement type, which rewrite-target pointers are bound:

| Statement | where | having | startwith | connectby | aftercb | merge_upd | merge_ins | merge_del | orderby_for | show_arg |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| SELECT | ✓ | ✓ | ✓ if present | ✓ if present | ✓ if present | | | | ✓ | ✓ if SHOWSTMT |
| UPDATE | ✓ | | | | | | | | ✓ | |
| DELETE | ✓ | | | | | | | | | |
| INSERT (with SELECT source) | ✓ | | | | | | | | | |
| MERGE | ✓ | | | | | ✓ | ✓ | ✓ | | |
| UNION/DIFFERENCE/INTERSECTION | ✓ (only if rewritten as derived) | | | | | | | | ✓ | |

Unbound pointers stay aimed at a local `dummy = NULL`, so dereferences inside Phase 2 are no-ops.

## Hot invariants

- `parser_walk_tree(qo_rewrite_queries, ...)` recurses into nested subqueries automatically. Each nested SELECT goes through phases 1–3 independently.
- `qo_reduce_equality_terms_post` runs as a post-walk so child queries reduce first; this gives the parent's correlation references the chance to see folded-constant subquery outputs.
- `pt_cnf` is idempotent — calling it on already-CNF input is a no-op.
- `done_reduce_equality_terms` flag prevents double-reduction of the same subtree.
- The `PT_EXPR_INFO_COPYPUSH` flag marks copy-pushed predicates that must be removed if their target spec was demoted from outer to inner.
- Auto-parameterize skips terms tagged `PT_EXPR_INFO_DO_NOT_AUTOPARAM` — see the auto-parameterize page for the rule set.

## Cross-references

- Calls into `query_rewrite_select.c`: `qo_analyze_path_join_pre`, `qo_analyze_path_join`, `qo_check_generate_single_tbl_connect_by`, `qo_rewrite_select_queries`, `qo_move_on_of_explicit_join_to_where`, `qo_rewrite_index_hints`, `qo_rewrite_nonnull_count_select_list` ([[components/optimizer-rewriter-select]]).
- Calls into `query_rewrite_term.c`: `qo_rewrite_terms`, `qo_reduce_equality_terms`, `qo_reduce_equality_terms_post` ([[components/optimizer-rewriter-term]]).
- Calls into `query_rewrite_subquery.c`: `qo_rewrite_subqueries`, `qo_rewrite_hidden_col_as_derived`, `qo_add_limit_clause` ([[components/optimizer-rewriter-subquery]]).
- Calls into `query_rewrite_set.c`: `qo_check_distinct_union`, `qo_check_hint_union`, `qo_push_limit_to_union` ([[components/optimizer-rewriter-set]]).
- Calls into `query_rewrite_auto_parameterize.c`: `qo_auto_parameterize`, `qo_auto_parameterize_limit_clause`, `qo_auto_parameterize_keylimit_clause` ([[components/optimizer-rewriter-auto-parameterize]]).
- Parser helpers: `pt_cnf`, `pt_split_join_preds`, `pt_limit_to_numbering_expr`, `pt_get_subquery_of_insert_select`, `pt_is_query`, `pt_has_non_groupby_column_node`, `pt_find_aggregate_functions_pre/post`, `pt_is_pseudocolumn_node`, `pt_is_const_not_hostvar`, `pt_rewrite_to_auto_param`.
- View-transform helper: `mq_rewrite_query_as_derived`.

## Smells / observations

- The `PT_UNION` arm contains a duplicated comment block (`:167-172`) — same paragraph repeated twice, inert but should be folded.
- `single_table_opt` flag is set once and never cleared — implicit one-shot. If a future code path re-enters the `PT_SELECT` arm (e.g. via re-walking after a structural rewrite), the early-out at `:73` would skip the connect-by split. Comment at `:65-71` documents this.
- Phase 3 runs `qo_auto_parameterize_limit_clause` even when the predicate-level gate is off — limit clauses are always parameterized when the global SQL-mode and skip flags allow. KEYLIMIT follows the same pattern.
- The `PROCESS_IF_EXISTS` macro deliberately evaluates `*condition` once into a function call. Misnamed — does NOT condition on existence of `func`; conditions on existence of the slot pointee.

## Related

- Parent: [[components/optimizer]]
- Sibling pages: [[components/optimizer-rewriter-select]], [[components/optimizer-rewriter-term]], [[components/optimizer-rewriter-subquery]], [[components/optimizer-rewriter-set]], [[components/optimizer-rewriter-auto-parameterize]], [[components/optimizer-rewriter-unused-function]]
- Pipeline: [[Query Processing Pipeline]] · [[components/parser]] · [[components/parse-tree]] · [[components/semantic-check]] · [[components/name-resolution]]
