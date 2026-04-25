---
type: component
parent_module: "[[components/optimizer-rewriter]]"
path: "src/optimizer/rewriter/query_rewrite_select.c"
status: active
purpose: "SELECT-shape mutations after CNF normalization: outer→inner join demotion, explicit→implicit inner-join flattening, OID-equality→derived-table rewrite, outer-joined-table elimination via unique constraints, parent-table elimination via PK/FK, ORDER BY pruning, USING-INDEX hint normalization, single-table CONNECT BY enablement, and COUNT(col)→COUNT(*) upgrade"
key_files:
  - "query_rewrite_select.c (3795 LOC)"
public_api:
  - "qo_rewrite_select_queries(parser, &node, wherep, &seqno) → bool — master driver for SELECT-shape rewrites"
  - "qo_rewrite_index_hints(parser, statement) — normalize/dedup/sort USING-INDEX hint list"
  - "qo_move_on_of_explicit_join_to_where(parser, fromp, wherep) — splice ON clauses onto WHERE for unified rewriting"
  - "qo_analyze_path_join_pre / qo_analyze_path_join — classify path-specs as INNER/OUTER/OUTER_WEASEL"
  - "qo_check_generate_single_tbl_connect_by(parser, node) → bool — CONNECT BY single-table optimization eligibility"
  - "qo_rewrite_nonnull_count_select_list(parser, select) — COUNT(col) → COUNT(*) for NOT-NULL columns"
tags:
  - component
  - cubrid
  - optimizer
  - rewriter
  - select
  - join
related:
  - "[[components/optimizer-rewriter]]"
  - "[[components/optimizer-rewriter-term]]"
  - "[[components/optimizer-rewriter-subquery]]"
  - "[[components/parse-tree]]"
created: 2026-04-25
updated: 2026-04-25
---

# `query_rewrite_select.c` — SELECT-shape Rewriter

3795 LOC. Owns mutations that change the **structural form** of a SELECT statement after CNF normalization but before plan generation: outer→inner join demotion, explicit→implicit join demotion, ordered/redundant ORDER BY reduction, OID-equality→derived-table rewrite, outer-joined-table elimination via unique constraints, parent-table elimination via PK/FK semantics, USING-INDEX hint normalization, single-table CONNECT BY enablement, and `COUNT(col)`→`COUNT(*)` upgrade.

## Role in the rewriter pipeline

```
mq_rewrite (query_rewrite.c:736)
  parser_walk_tree(qo_rewrite_queries, qo_rewrite_queries_post)
    qo_rewrite_queries (per-node, query_rewrite.c:43)
      [pre-rewrite, PT_SELECT branch]
      qo_move_on_of_explicit_join_to_where     (this file, :515)
      qo_rewrite_index_hints                   (this file, :150)
      parser_walk_tree(qo_analyze_path_join_pre, qo_analyze_path_join)
                                               (this file, :559 / :597)
      qo_rewrite_nonnull_count_select_list     (this file, :3776)
      [optimization phase, after CNF + reduce_equality + rewrite_terms]
      qo_rewrite_select_queries                (this file, :61)
```

Distinguishing what this file is **not**: it is not a CNF rewriter (that's [[components/optimizer-rewriter-term]]), not a subquery flattener (that's [[components/optimizer-rewriter-subquery]]), not the auto-parameterizer ([[components/optimizer-rewriter-auto-parameterize]]). It is specifically the layer that mutates SELECT shape — join types, ORDER BY shape, derived-table introductions, table elimination, hint shape — after the term layer has normalized predicates.

## Public surface (all consumed only inside `src/optimizer/rewriter/`)

| Function | Line | Role |
|---|---:|---|
| `qo_rewrite_select_queries` | 61 | Master driver: outer→inner, inner-flatten, OID rewrite, ORDER BY reduction, sargable predicate push, outer-joined-table elimination, FK-PK table elimination. Returns `false` on giveup. |
| `qo_rewrite_index_hints` | 150 | Normalize/dedup/sort `USING INDEX` hint list for SELECT/UPDATE/DELETE. |
| `qo_move_on_of_explicit_join_to_where` | 515 | Splice all `ON` clauses onto end of WHERE list (location-tagged) for unified rewrite; restored by `qo_rewrite_queries_post`. |
| `qo_analyze_path_join_pre` | 559 | parser_walk pre-pruner for path-spec analysis. |
| `qo_analyze_path_join` | 597 | Tag path-specs as `PT_PATH_INNER` / `PT_PATH_OUTER` / `PT_PATH_OUTER_WEASEL`. |
| `qo_check_generate_single_tbl_connect_by` | 3633 | Predicate: can a `CONNECT BY` query use single-table heap/index scan optimization. |
| `qo_rewrite_nonnull_count_select_list` | 3776 | `COUNT(col)` → `COUNT(*)` when `col` has NOT-NULL/PK constraint and no filter index. |

External call-site grep returned **none** — every public symbol is consumed inside the rewriter directory. The header `query_rewrite.h:20` says "don't include this except files in this folder".

## Major rewrites

### Outer → inner join demotion (`qo_rewrite_outerjoin`, :3391)

For each outer-joined spec in `from` (`mq_is_outer_join_spec`), walks WHERE looking for top-level (location 0, no `or_next`, not AND/OR/IS_NULL) predicates that (a) reference the spec and (b) are non-nullable for it (`qo_check_nullable_expr_with_spec`). If found, `LEFT_OUTER` becomes `INNER`, location is reset on the WHERE via `qo_reset_location`, and connected `RIGHT_OUTER` siblings up to the next `PT_JOIN_NONE` are also flipped to `INNER`. Iterates with `rewrite_again` until fixpoint. **Skipped under `connect_by`** because hierarchical query semantics consume join output.

### Explicit → implicit inner-join flatten (`qo_rewrite_innerjoin`, :3530)

Within each disconnected spec group (delimited by `PT_JOIN_NONE`) that contains no outer joins, demotes `PT_JOIN_INNER` to `PT_JOIN_NONE`, releasing the optimizer to pick its own join order. Skipped if `PT_HINT_ORDERED` is set (`/*+ ORDERED */`) or `connect_by` exists. Recurses into derived `PT_IS_SUBQUERY` tables.

### OID-equality → derived-table-of-set rewrite (`qo_rewrite_oid_equality`, :1059)

Targets predicates of shape `x = expr` or `x IN (…)` where `x` is `PT_OID_ATTR` and the RHS is OID-constant (`qo_is_oid_const :791`: `PT_VALUE`, `PT_HOST_VAR`, `PT_PARAMETER`, set-functions of constants, uncorrelated subquery). Replaces

```
SELECT … FROM c x, … WHERE … AND x = expr
→
SELECT … FROM TABLE({expr}) AS t(x), … WHERE …
```

Attribute references `x.i` are converted to `t.x.i` via `qo_convert_attref_to_dotexpr`. Skipped when `meta_class == PT_META_CLASS`. After rewrites fire, `qo_analyze_path_join*` is rerun.

Helper chain: `qo_get_next_oid_pred (:760)` → `qo_construct_new_set (:866)` → `qo_make_new_derived_tblspec (:945)` → `qo_convert_attref_to_dotexpr_pre/_post (:654/:681)` → `mq_reset_ids_in_statement`.

### Outer-joined-table elimination via unique-key coverage (`qo_reduce_outer_joined_tbls`, :1908)

For a `LEFT_OUTER` spec with `PT_ONLY` (no inheritance), no `PT_HINT_NO_ELIMINATE_JOIN`, and no references outside its `on_cond`: if the `ON`-clause `=`-predicates against the spec **cover all attributes** of any unique-family constraint (`SM_IS_CONSTRAINT_UNIQUE_FAMILY`), the spec and its `on_cond` predicates are excised.

```sql
-- before
SELECT COUNT(*) FROM t1 a LEFT OUTER JOIN t2 b ON a.pk = b.pk
-- after (b dropped, ON-clause dropped)
SELECT COUNT(*) FROM t1 a
```

> [!warning] Right-outer not supported
> Comment at :1921: `/* TO_DO : for right outer join */`. Right-outer joins never trigger this elimination.

### Parent-table elimination via PK/FK semantics (`qo_reduce_joined_tbls_ref_by_fk`, :2104)

Drops a parent (PK-side) spec joined to a child (FK-side) spec when:

- both are `PT_JOIN_NONE` / `INNER` / `NATURAL` (not outer/cross/full/union)
- both resolve to physical classes (no CTE/derived)
- the PK constraint's `fk_info` is non-null
- every referenced attr of the parent in WHERE participates in a join `=` predicate or a reducible non-join `=` predicate (matching constants are stringified via `parser_print_tree` with `PT_CONVERT_RANGE` and case-insensitively compared)
- all PK columns are covered (`cons_attr_flag` bitmap)
- the parent OID equals `fk_info->ref_class_oid` (so parent must be the *exact* class, not a parent in the inheritance hierarchy)
- the parent is not the head of a left/right/full-outer next sibling

When eliminated, all join `=` predicates and matched parent-side equality predicates are deleted, and `IS NOT NULL` predicates are appended for any FK columns lacking `SM_ATTFLAG_NON_NULL`.

Helper chain: `qo_check_pk_ref_by_fk_in_parent_spec (:2322)` → `qo_check_fks_ref_pk_in_child_spec (:2629)` → `qo_check_fk_ref_pk_in_child_spec (:2756)` → `qo_check_reduce_predicate_for_parent_spec (:2964)` → `qo_reduce_predicate_for_parent_spec (:3162)`.

> [!warning] String-rendered constant comparison
> Constant matching uses `pt_str_compare(parser_print_tree(...), ..., CASE_INSENSITIVE)`. Two constants that *evaluate* equal but render differently (e.g. `1` vs `1.0`, locale-dependent decimal) defeat this optimization.

### USING INDEX hint normalization (`qo_rewrite_index_hints`, :150)

Four phases:

1. **`USING INDEX NONE` short-circuit** (:186) — drop everything except the NONE node.
2. **`USING INDEX ALL EXCEPT` purge of `t.none`** (:224) — bare table-level `t.none` items are removed (nonsensical inside ALL EXCEPT).
3. **Bubble-sort + dedup** (:269) — sort by `PT_IDX_HINT_ORDER` priority (`CLASS_NONE > IGNORE > FORCE > USE`), then by table name, then index name; collapse exact duplicates; for keylimit-aware dedup, prefer the variant with `indx_key_limit`.
4. **Mask removal** (:392) — `{IGNORE,FORCE} idx` on `t` masks lower-priority `idx` on the same table; `t.none` masks any hint on `t`.

### ORDER BY pruning + ORDERBY_NUM lambda swap (`qo_reduce_order_by` + `qo_reduce_order_by_for`, :1263 / :1179)

- If `GROUP BY` covers `ORDER BY` (`pt_sort_spec_cover`) and there is no DISTINCT/HAVING/`orderby_num()` in the select-list, ORDER BY is **dropped wholesale** and `orderby_for` is rewritten via `pt_lambda_with_arg` to substitute `ORDERBY_NUM()` → `GROUPBY_NUM()` and migrated into HAVING.
- Constant-marked select-list expressions (`PT_NAME_INFO_CONSTANT`) are dropped from ORDER BY.
- Duplicate ORDER positions reuse left-most occurrence index; duplicates with conflicting asc/desc raise `MSGCAT_SEMANTIC_SORT_DIR_CONFLICT`.
- After reduction, if all ordering is gone, `ORDERBY_NUM()` in `orderby_for` is rewritten to `INST_NUM()` and the predicate is appended into WHERE; otherwise to `GROUPBY_NUM()` if the discarded sort had `orderby_num()` over GROUP BY; otherwise the select-list reference of `ORDERBY_NUM()` is rewritten to `INST_NUM()`.
- Skipped entirely under `connect_by`.

The two-phase merge-check at :1314 and :1454 re-runs the GROUP-BY-covers-ORDER-BY test after constant pruning; without the second pass, dropping a constant ORDER BY column could newly satisfy coverage missed initially.

### Sargable-predicate copy-push into derived tables

`qo_rewrite_select_queries:107-119` calls `mq_copypush_sargable_terms` (in `parser/view_transform.c`) for every spec whose `derived_table_type` is `PT_IS_SUBQUERY` or `PT_DERIVED_DBLINK_TABLE`. Pushed terms are tagged `PT_EXPR_INFO_COPYPUSH`; `qo_rewrite_queries_post` later (in `query_rewrite.c:670`, :702) frees those tagged copies after they've been re-located to ON-clauses.

### Path-spec join classification (`qo_analyze_path_join`, :597)

For each `PT_SPEC` carrying `path_conjuncts` (path expression `c.attr` walking through OID), classifies as:

| Tag | Meaning |
|---|---|
| `PT_PATH_INNER` | null path → no row produced |
| `PT_PATH_OUTER` | null path → no WHERE effect |
| `PT_PATH_OUTER_WEASEL` | might affect WHERE; treated as outer but tagged so optimizer cannot merge with inner |

Sub-paths are processed first (post-order); a single `PT_PATH_INNER` child taints upward via `qo_find_best_path_type (:475)`.

### `COUNT(col)` → `COUNT(*)` upgrade (`qo_rewrite_nonnull_count*`, :3701 / :3776)

Triggers when a `PT_COUNT(PT_ALL, name)` references a non-outer-joined spec whose class has a NOT-NULL-family constraint (`SM_IS_CONSTRAINT_NOT_NULL_FAMILY`) covering the column — provided no constraint on the class has a `filter_predicate`. Aborts if any spec in `from` is `RIGHT_OUTER`, `FULL_OUTER`, or `CROSS`.

### Single-table CONNECT BY enablement (`qo_check_generate_single_tbl_connect_by`, :3633)

Predicate consulted by `qo_rewrite_queries:87`. Allows the rewriter to fold `START WITH` into WHERE and set `single_table_opt = 1` for a hierarchical query that has:

- no joins
- no WHERE
- no method calls in select-list
- not on a class hierarchy (`PT_ONLY`)
- not partitioned
- optimization on

### ON-clause / WHERE-clause shuttle (`qo_move_on_of_explicit_join_to_where`, :515)

ON conjuncts are appended to WHERE before rewrite; each retains a positive `info.expr.location` matching its owning spec's `info.spec.location`. After rewrites mutate join types, `qo_rewrite_queries_post` (in `query_rewrite.c:585`) splices each tagged term back to the right spec's `on_cond` — **unless** the spec was demoted to inner or none, in which case the term keeps `location = 0` and stays in WHERE. `qo_reset_location (:3494)` clears location tags within a range when an outer→inner demotion happens.

## Hot / tricky invariants

- **CNF assumed.** Every WHERE/HAVING is in conjunctive normal form by the time these functions run. Many helpers explicitly bail when `or_next != NULL` (e.g. `qo_collect_name_with_eq_const :1772`, `qo_check_pk_ref_by_fk_in_parent_spec :2420`). Disjunctions are deliberately not eliminated.
- **Location tagging is the load-bearing channel.** ON-clause terms get `info.expr.location > 0` at parse-time. The rewriter promotes them to WHERE for unified processing, then `qo_rewrite_queries_post` walks WHERE and pushes any term with `location > 0` back into `spec->info.spec.on_cond` of the spec sharing that location. Mutating a spec's `location` requires calling `qo_reset_spec_location (:1862)` to walk WHERE and renumber predicates.
- **`PT_EXPR_INFO_COPYPUSH` is a lifetime marker.** Sargable terms copy-pushed into derived tables are marked; if `qo_rewrite_queries_post` finds one already in the (final) WHERE, it free-trees it.
- **Connect-by killswitch.** `qo_rewrite_outerjoin :3404`, `qo_rewrite_innerjoin :3541`, and `qo_reduce_order_by :1275` early-return if `connect_by != NULL`. Reason: hierarchical query semantics put WHERE *after* HQ evaluation; mutating join types or constant-folding ORDER BY before HQ would change observable rows or sort.
- **Unique-key coverage uses an attribute-bitmap.** `cons_attr_flag = (1 << i) - 1` (:2478, :2843, :3063) — implicitly limits any composite key with > 32 columns. Nothing in the code guards that boundary; it's an undocumented ceiling.
- **`qo_rewrite_oid_equality` mutates the WHERE list while iterating.** The driver loop :79-90 saves the successor pointer in `next` so a freed/relocated `pred` doesn't cause a use-after.
- **Idempotency.** Not formally idempotent. Reruns can fire if `qo_rewrite_oid_equality` succeeds (path-join reanalysis is forced at :93), but each individual rewrite is structurally guarded so a second invocation is a no-op in practice.

## Internal data structures (declared in `query_rewrite.h`, used here)

- `SPEC_ID_INFO {id, appears, nullable}` — spec-presence/nullability probe in `qo_analyze_path_join`, `qo_rewrite_outerjoin`.
- `SPEC_CNT_INFO {spec, my_spec_cnt, other_spec_cnt, my_spec_node}` — counters used by reduction logic.
- `TO_DOT_INFO {old_spec, new_spec}` — old→new spec swap context for OID-equality rewrite.
- `RESET_LOCATION_INFO {start_spec, start, end, found_outerjoin}` — range-tagged location resetter.
- `QO_REDUCE_REFERENCE_INFO` (8 fields incl. PK/FK MOPs, constraints, exclude-lists, predicate point-lists) — driver context for the FK-PK reduction state machine.

The two enums in the header (`COMP_DBVALUE_WITH_OPTYPE_RESULT`, `DNF_MERGE_RANGE_RESULT`) are declared but **not used** in this file — they belong to [[components/optimizer-rewriter-term]].

No file-scope statics, no module-private lookup tables.

## Static helpers worth naming

| Helper | Line | Purpose |
|---|---:|---|
| `qo_find_best_path_type` | 475 | Recursive path-type promotion (any INNER child taints upward). |
| `qo_convert_attref_to_dotexpr_pre/_post` | 654 / 681 | Pruner + in-place transmute `PT_NAME(x)` → `PT_DOT_(t,x)`. |
| `qo_get_next_oid_pred` | 760 | Find next CNF term that is `PT_OID_ATTR = expr` or `IN expr`. |
| `qo_is_oid_const` | 791 | Recognize "constant for OID rewrite". |
| `qo_construct_new_set` | 866 | Build `F_SEQUENCE` wrapping the constant side of an OID predicate. |
| `qo_make_new_derived_tblspec` | 945 | Wrap class spec into derived `TABLE({…}) AS t(x)`. |
| `qo_reduce_order_by_for` | 1179 | Lambda-substitute `ORDERBY_NUM()` → `GROUPBY_NUM()` and shift to HAVING. |
| `qo_get_name_cnt_by_spec` | 1626 | Count name refs to spec (early stop). |
| `qo_get_name_cnt_keep_unique` | 1659 | Same, but only walks expressions preserving uniqueness. |
| `qo_get_name_cnt_by_spec_no_on` | 1709 | Count refs to spec, ignoring its own `on_cond`. |
| `qo_collect_name_with_eq_const` | 1759 | Collect `spec.col = (something not referencing spec)` predicates from ON. |
| `qo_modify_location` | 1830 | In-place re-tag location across `PT_EXPR/NAME/VALUE`. |
| `qo_reset_spec_location` | 1862 | Shift each spec's location down by one and propagate to WHERE. |
| `qo_reset_location` | 3494 | Zero `location` for `PT_EXPR/NAME/VALUE` in [start, end]. |
| `qo_is_exclude_spec` | 2282 | Point-list contains test (set membership over `PT_NODE_POINTER` lists). |

## Smells / known limitations

- **Right-outer not handled** in `qo_reduce_outer_joined_tbls` — TODO comment at :1921.
- **String-rendered constant comparison** in FK reduction — locale or formatting differences defeat the optimization.
- **32-column composite-key ceiling** baked into `cons_attr_flag = (1 << i) - 1`; undocumented.
- **Copy-pasted bitmap loops** at :2470-2541, :2836-2927, :3063-3072 — consolidation opportunity (pull into a `pk_attr_index_lookup` helper).
- **Dead conditional at :705** inside `qo_convert_attref_to_dotexpr` (`if (meta_class == PT_OID_ATTR)` is already inside `case PT_OID_ATTR`).
- **Subtle control flow at :3125-3129** in `qo_check_reduce_predicate_for_parent_spec` (`if (child_pred_point_list == NULL)` test inside parent-loop after running inner search loop).
- **Redundant declarations** at :1190-1191 (`ord_num`/`grp_num` declared then immediately re-NULLed).
- **No domain-specific `ER_*` codes** raised here — only `ER_GENERIC_ERROR` via `er_set` at :1248, :1613, plus semantic messages via `PT_ERRORm`/`PT_ERRORmf`.

## Cross-references

- **`pt_*` parser helpers** (high-volume): `pt_cnf`, `pt_point`, `pt_name`, `pt_lambda_with_arg`, `pt_value_to_db`, `pt_get_end_path_node`, `pt_name_equal`, `pt_find_order_value_in_list`, `pt_remove_from_list`, `pt_to_null_ordering`, `pt_sort_spec_cover`, `pt_check_orderbynum_pre/post`, `pt_user_specified_name_compare`, `pt_str_compare`, `pt_short_print`, `pt_is_attr`, `pt_is_expr_node`, `pt_is_value_node`, `pt_expr_keep_uniqueness`, `pt_find_entity`, `parser_walk_tree`, `parser_copy_tree(_list)`, `parser_new_node`, `parser_free_node/tree`, `parser_append_node`, `parser_make_expression`, `parser_print_tree`.
- **Sibling `qo_*`** (in `query_rewrite_term.c`): `qo_check_condition_null`, `qo_check_nullable_expr_with_spec`, `qo_get_name_by_spec_id`, `qo_is_reduceable_const`.
- **`mq_*`** (in `parser/view_transform.c`): `mq_copypush_sargable_terms`, `mq_is_outer_join_spec`, `mq_generate_name`, `mq_reset_ids_in_statement`, `mq_rewrite_query_as_derived`.
- **Schema manager `sm_*`**: `sm_class_constraints`, `sm_find_class`, `sm_is_partitioned_class`.
- **Internationalization**: `intl_identifier_casecmp`.
- **Semantic messages**: `MSGCAT_SEMANTIC_OUT_OF_MEMORY` (:1203, :1404, :1493, :1541, :1578), `MSGCAT_SEMANTIC_SORT_DIR_CONFLICT` (:1440).

## Related

- Parent: [[components/optimizer-rewriter]]
- Sibling: [[components/optimizer-rewriter-term]] (CNF + term reduction), [[components/optimizer-rewriter-subquery]] (subquery flattening), [[components/optimizer-rewriter-set]] (UNION pushdown).
- [[Query Processing Pipeline]]
