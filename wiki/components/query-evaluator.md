---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/query_evaluator.c"
status: active
purpose: "Predicate evaluation hub: eval_pred recursion over PRED_EXPR trees, three-valued logic (V_TRUE/V_FALSE/V_UNKNOWN/V_ERROR), data-filter and key-filter entry points, set/list comparisons"
key_files:
  - "query_evaluator.c"
  - "query_evaluator.h"
tags:
  - component
  - cubrid
  - query
  - server
  - predicate
related:
  - "[[components/query|query]]"
  - "[[components/query-opfunc|query-opfunc]]"
  - "[[components/query-string|query-string]]"
  - "[[components/query-regex|query-regex]]"
  - "[[components/xasl-predicate|xasl-predicate]]"
  - "[[components/filter-pred-cache|filter-pred-cache]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/db-value|db-value]]"
created: 2026-04-23
updated: 2026-04-23
---

# `query_evaluator.c` — Predicate Evaluation Hub

Server/SA-only module. Evaluates `PRED_EXPR` boolean trees against individual tuples using three-valued logic (True / False / Unknown / Error). Provides the `eval_pred` recursive core plus top-level entry points `eval_data_filter` (heap filter) and `eval_key_filter` (index key filter).

## Purpose

Every WHERE predicate, HAVING clause, join condition, and filtered-index predicate in CUBRID is represented as a `PRED_EXPR` tree (see [[components/xasl-predicate|xasl-predicate]]). `query_evaluator.c` walks this tree recursively and returns a `DB_LOGICAL` result. It is the lowest-level evaluation layer: it calls `fetch_peek_dbval` (via `fetch.c`) to resolve operand values, and delegates arithmetic/comparison to `tp_value_compare_with_error` and string matching to `db_string_like` / `db_string_rlike`.

## Public Entry Points

| Signature | Role |
|-----------|------|
| `eval_pred(thread_p, pr, vd, obj_oid)` | Main predicate evaluator — recursive over PRED_EXPR tree; returns `DB_LOGICAL` |
| `eval_data_filter(thread_p, oid, recdes, scan_cache, filter)` | Heap tuple filter: materialise attributes, then call `eval_pred` via the predicate's cached function pointer |
| `eval_key_filter(thread_p, value, prefix_size, prefix_value, filter)` | Index key filter: evaluate predicate against key DB_VALUE before fetching the heap record |
| `eval_fnc(thread_p, pr, single_node_type)` | Returns the specialised `PR_EVAL_FNC` for a given PRED_EXPR shape (avoids the generic switch at execution time) |
| `update_logical_result(thread_p, ev_res, qualification)` | Apply `QPROC_QUALIFIED / NOT_QUALIFIED / QUALIFIED_OR_NOT` to a DB_LOGICAL result |
| `eval_pred_comp0 / comp1 / comp2 / comp3` | Specialised evaluators for comparison predicates (0 = no collation, 1 = with collation, etc.) |
| `eval_pred_alsm4 / alsm5` | SOME/ALL quantified comparison evaluators |
| `eval_pred_like6` | LIKE predicate evaluator |
| `eval_pred_rlike7` | RLIKE/REGEXP predicate evaluator — calls `db_string_rlike` with the XASL node's `comp_regex` cache |

## Execution Path

```
scan_manager.c: scan_next_scan()
  → eval_data_filter(thread_p, oid, recdes, scan_cache, filter)
      → heap_attrinfo_read_dbvalues(thread_p, oid, recdes, scan_cache, attr_cache)
      → eval_pred(thread_p, filter->scan_pred->pred_expr, vd, oid)

eval_pred(thread_p, pr, vd, obj_oid):
  switch (pr->type)
    T_PRED:
      B_AND → right-linear tree loop; short-circuit on V_FALSE
      B_OR  → right-linear tree loop; short-circuit on V_TRUE
      B_XOR → evaluate both sides; result = (lhs != rhs)
      B_IS / B_IS_NOT → compare DB_LOGICAL results directly
      B_NOT → eval_negative(eval_pred(pr->rhs))

    T_EVAL_TERM:
      ET_COMP → fetch lhs+rhs, eval_value_rel_cmp(dbval1, dbval2, rel_op, et_comp)
      ET_ALSM → fetch item, set/list, eval_some_eval / eval_all_eval
      ET_LIKE → fetch src+pattern+esc, db_string_like()
      ET_RLIKE → fetch src+pattern+case_sens, db_string_rlike(&comp_regex, ...)

    T_NOT_TERM → eval_negative(eval_pred(pr->pe.m_not_term.arg))
```

## Three-Valued Logic (`DB_LOGICAL`)

> [!key-insight] Four values, not three
> CUBRID's `DB_LOGICAL` is actually **four-valued**: `V_FALSE=0`, `V_TRUE=1`, `V_UNKNOWN=2`, `V_ERROR=3`. `V_UNKNOWN` represents SQL NULL propagation (any comparison with NULL). `V_ERROR` is a hard engine error distinct from NULL-unknown. Callers must handle both `V_FALSE` and `V_UNKNOWN` as "not qualified" for filtering (standard SQL semantics: only V_TRUE passes a WHERE clause).

`eval_negative` maps `V_TRUE↔V_FALSE`, passes `V_UNKNOWN` and `V_ERROR` unchanged — correct SQL NOT semantics.

`eval_logical_result` (AND combiner): `V_ERROR` dominates; `V_FALSE` dominates over `V_UNKNOWN`; `V_UNKNOWN` dominates over `V_TRUE`.

### AND Short-Circuit

> [!key-insight] AND tree is right-linear and short-circuits on V_FALSE (not on V_UNKNOWN)
> The predicate generator in `xasl_generation.c` (`pt_to_pred_expr()`) constructs right-linear AND trees. `eval_pred` iterates the right spine via a `for` loop (not mutual recursion), exiting immediately on `V_FALSE` or `V_ERROR`. However, a `V_UNKNOWN` result from the LHS does **not** short-circuit; the loop must continue to check if a later term is `V_FALSE` (which would make the AND definitely false). The code preserves this with careful `V_UNKNOWN` tracking.

```c
// AND: short-circuit on V_FALSE only
for (t_pr = pr; result == V_TRUE && t_pr->type == T_PRED && t_pr->pe.m_pred.bool_op == B_AND;
     t_pr = t_pr->pe.m_pred.rhs)
{
  result = eval_pred(thread_p, t_pr->pe.m_pred.lhs, vd, obj_oid);
  if (result == V_FALSE || result == V_ERROR) goto exit;
  // V_UNKNOWN is preserved by the loop guard failing next iteration
}
```

### OR Short-Circuit

OR mirrors AND: short-circuits on `V_TRUE`; preserves `V_UNKNOWN` in case a later term is definite false.

## Comparison Evaluation (`eval_value_rel_cmp`)

Resolves two `DB_VALUE*` operands and calls `tp_value_compare_with_error(dbval1, dbval2, do_coerce, ...)`. A constant RHS is coerced at the first call and the coerced value is kept in-place (guarded by `REGU_VARIABLE_FETCH_ALL_CONST` flag) so subsequent row evaluations skip the coercion:

```c
if (REGU_VARIABLE_IS_FLAGED(et_comp->rhs, REGU_VARIABLE_FETCH_ALL_CONST))
{
  tp_value_auto_cast(peek_value_p, peek_value_p, regu_var->domain);
  // value now typed correctly; subsequent calls skip the cast
}
```

`R_NULLSAFE_EQ` (the `<=>` operator) is the only relational op that does not return `V_UNKNOWN` on NULL: NULL `<=>` NULL = TRUE, NULL `<=>` non-NULL = FALSE.

## LIKE Evaluation

`ET_LIKE` predicates call `db_string_like(src, pattern, esc, &result)` from [[components/query-string|query-string]], which dispatches to `qstr_eval_like`. The escape character is fetched from the XASL node's `et_like.esc_char`.

## RLIKE Evaluation

`ET_RLIKE` predicates call `db_string_rlike(src, pattern, case_sensitive, &comp_regex, &result)`, passing the **mutable cache pointer** stored in `RLIKE_EVAL_TERM::comp_regex`. See [[components/query-regex|query-regex]] for the compile-once caching semantics.

## Set and List-File Comparisons

`eval_set_list_cmp` handles IN/SOME/ALL predicates where an operand is a `TYPE_LIST_ID` (subquery result). It lazy-sorts the list file (via `qfile_sort_list`) on first evaluation, then calls the appropriate `eval_*_sort_list_to_*` helper. The sorted flag is set on the `QFILE_SORTED_LIST_ID` to avoid re-sorting.

## `FILTER_INFO` / `SCAN_PRED` Structures

```c
struct filter_info {
  SCAN_PRED *scan_pred;    // predicate + attr cache + eval function pointer
  SCAN_ATTRS *scan_attrs;  // attr ID array + heap cache
  val_list_node *val_list;
  VAL_DESCR *val_descr;
  OID *class_oid;
  // index key filter fields...
};
```

`eval_data_filter` uses `SCAN_PRED::pr_eval_fnc` (function pointer pre-computed by `eval_fnc()`) to skip the generic `switch` on the hot path.

## Integration with `filter-pred-cache`

When filtered indexes (`CREATE INDEX ... WHERE expr`) are used, the predicate is compiled once and cached in the [[components/filter-pred-cache|filter-pred-cache]]. The cache returns a `FILTER_INFO` with an already-built `SCAN_PRED`. `eval_key_filter` uses this directly — no XASL context is needed.

## Recursion Depth Guard

`eval_pred` tracks recursion depth via `thread_inc/dec_recursion_depth(thread_p)` and aborts with `ER_MAX_RECURSION_SQL_DEPTH` if it exceeds `PRM_ID_MAX_RECURSION_SQL_DEPTH`. This guards against stack overflow in deeply nested predicate trees (e.g., `a = 1 AND b = 2 AND ... AND z = 26`).

## Constraints

### NULL Propagation
Any NULL operand in a comparison yields `V_UNKNOWN`. The AND/OR short-circuit rules preserve SQL three-valued semantics: `FALSE AND UNKNOWN = FALSE`, `TRUE AND UNKNOWN = UNKNOWN`, `TRUE OR UNKNOWN = TRUE`, `FALSE OR UNKNOWN = UNKNOWN`.

### Memory Ownership
`eval_data_filter` calls `heap_attrinfo_read_dbvalues` which fills attribute values into cached `DB_VALUE` slots owned by the `SCAN_ATTRS::attr_cache`. These are `PEEK` mode — the pointers point into the page buffer and must not be freed.

### Threading
SERVER/SA-mode only. All state is on the call stack or in per-query XASL nodes. Thread-safe as long as the XASL node is not shared across threads (parallel execution clones the XASL).

### Error Model
Returns `V_ERROR` and sets `er_set` for hard errors (allocation failure, REGU_VARIABLE fetch failure, exceeding recursion limit). Returns `V_UNKNOWN` for NULL-induced unknowns.

## Lifecycle

- Called per-tuple during scans.
- `eval_fnc` is called once per predicate node at scan open time to cache the function pointer.
- No inter-call state (except the regex `comp_regex` cache which lives in the XASL node).

## Related

- [[components/xasl-predicate|xasl-predicate]] — PRED_EXPR / COMP_EVAL_TERM / LIKE_EVAL_TERM / RLIKE_EVAL_TERM
- [[components/filter-pred-cache|filter-pred-cache]] — supplies pre-compiled FILTER_INFO for filtered indexes
- [[components/scan-manager|scan-manager]] — calls `eval_data_filter` and `eval_key_filter`
- [[components/query-string|query-string]] — LIKE and REGEXP predicates call `db_string_like` / `db_string_rlike`
- [[components/query-regex|query-regex]] — RLIKE compiled-pattern caching
- [[components/regu-variable|regu-variable]] — REGU_VARIABLE operand fetching
- [[components/query|query]] — hub
