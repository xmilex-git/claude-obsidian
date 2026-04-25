---
type: component
parent_module: "[[components/optimizer-rewriter]]"
path: "src/optimizer/rewriter/query_rewrite_term.c"
status: active
purpose: "Predicate-term rewrites operating on CNF-form WHERE/HAVING/START-WITH/CONNECT-BY/MERGE-WHERE: equality propagation with transitive-join inference, sargable normalization, BETWEEN/range/IN merging, LIKE simplification, IS NULL/NOT NULL folding, and outer-join-safe empty-conjunct collapse"
key_files:
  - "query_rewrite_term.c (4294 LOC) — the largest file in the rewriter"
public_api:
  - "qo_rewrite_terms(parser, nodes, terms) — orchestrates the 6-stage term pipeline"
  - "qo_reduce_equality_terms(parser, node, wherep) — equality propagation + transitive-join inference"
  - "qo_reduce_equality_terms_post — post-walk variant for SELECT subtrees"
  - "qo_get_name_by_spec_id — walker: did any PT_NAME match this spec_id?"
  - "qo_check_nullable_expr — count expressions whose result can be non-NULL despite NULL input"
  - "qo_check_nullable_expr_with_spec — same, scoped to a spec"
  - "qo_check_condition_null(parser, path_spec, query_where) → bool — outer→inner probe"
  - "qo_is_reduceable_const — treat CAST/TO_ENUMERATION_VALUE chains around a host-var as constant-equivalent"
tags:
  - component
  - cubrid
  - optimizer
  - rewriter
  - cnf
  - dnf
  - range
  - like
  - selectivity
related:
  - "[[components/optimizer-rewriter]]"
  - "[[components/optimizer-rewriter-select]]"
  - "[[components/optimizer-rewriter-auto-parameterize]]"
  - "[[components/parser]]"
  - "[[components/parse-tree]]"
created: 2026-04-25
updated: 2026-04-25
---

# `query_rewrite_term.c` — Predicate-Term Rewrites

4294 LOC — the largest single file in the rewriter. Owns the per-term shape mutations that happen on a **CNF-form** predicate list (WHERE / HAVING / START WITH / CONNECT BY / after-CB filter / MERGE update-insert-delete WHERE). The CNF transformation itself happens in `pt_cnf` (called from the orchestrator at `query_rewrite.c:316-351`); this file consumes the result.

> [!note] File-header drift
> The file's outer comment block at line 20 calls itself `query_rewrite_predicate.c` — historical name from before the rename. The actual file is `query_rewrite_term.c`. The enum `comp_dbvalue_with_optye_result` (header `query_rewrite.h:100-109`) carries an even older typo (`optye` for `optype`).

## What this file owns

- **Equality reduction / constant propagation** with derived-table lambda substitution and transitive-join inference (`qo_reduce_equality_terms`, :448).
- **Sargable-form normalization** — attribute on LHS, paired UNARY_MINUS folding, PRIOR hoisting (`qo_converse_sarg_terms`, :1035).
- **Pair-merge** — `attr op c1 AND attr op c2` → `BETWEEN` (`qo_reduce_comp_pair_terms`, :1702).
- **LIKE simplification** — `LIKE` → `IS NOT NULL` / `=` / `BETWEEN(GE_LT, lo, hi)` / index-scannable `BETWEEN(LIKE_LOWER_BOUND, LIKE_UPPER_BOUND)` (`qo_rewrite_like_terms`, :2456 and helpers).
- **Range conversion** — `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN` → `PT_RANGE` carrying a DNF list of `PT_BETWEEN_*` sub-ranges (`qo_convert_to_range`, :3526).
- **Range merging** within a single `PT_RANGE` DNF list (DNF-OR-union semantics) (`qo_merge_range_helper`, :3103).
- **Range intersection** across two CNF terms on the same attribute (CNF-AND semantics) (`qo_apply_range_intersection`, :4038).
- **`IS NULL` / `IS NOT NULL` folding** when sibling proves non-null, plus column NOT-NULL constraint awareness (`qo_fold_is_and_not_null`, :1445).
- **Trivial `v = v` removal** — final pass at end of `qo_reduce_equality_terms` (lines 927-973).
- **Outer-join-safe empty-conjunct collapse** — `PT_VALUE 0` falsification, location-scoped to one ON clause vs whole WHERE.
- **Path-spec NULL substitution** for outer-join elimination probing (`qo_check_condition_null` + `qo_replace_spec_name_null`).

What it does NOT own: NOT-pushdown / De Morgan (in `pt_cnf`), subquery rewrites (in [[components/optimizer-rewriter-subquery]]), DISTINCT / UNION rewrites (in [[components/optimizer-rewriter-set]]), `x AND TRUE` removal as such (handled implicitly via constant folding in `pt_semantic_type` and the `1=1` cut-off).

## Pipeline

`qo_rewrite_terms (parser, spec, terms)` (line 46) is the single fixed entry. It runs **conditionally on `*terms != NULL`** (line 48). The 6 stages execute in **strict order**:

```
qo_rewrite_terms()
├── qo_converse_sarg_terms       :1035 — attr-on-LHS, fold ±, hoist PRIOR
├── qo_reduce_comp_pair_terms    :1702 — pair → BETWEEN
├── qo_rewrite_like_terms        :2456 — LIKE → EQ / IS NOT NULL / BETWEEN
├── qo_convert_to_range          :3526 — = / < / <= / > / >= / BETWEEN / IN → PT_RANGE
├── qo_apply_range_intersection  :4038 — ANDed PT_RANGEs on same attr
└── qo_fold_is_and_not_null      :1445 — IS [NOT] NULL eliminator
```

`qo_reduce_equality_terms` runs **earlier** in the orchestrator (`query_rewrite.c:427-448`) **before** `qo_rewrite_terms`. So the actual term-pipeline ordering observed by a SELECT is:

1. `pt_cnf` (in `query_rewrite.c:318`)
2. `qo_reduce_equality_terms_post` (post-order tree walk; `query_rewrite.c:442`)
3. `qo_rewrite_terms` (the 6 stages above, `query_rewrite.c:476`)
4. `qo_rewrite_select_queries` ([[components/optimizer-rewriter-select]])
5. `qo_auto_parameterize` ([[components/optimizer-rewriter-auto-parameterize]])

## Public surface

| Function | Line | Role | External callers |
|---|---:|---|---|
| `qo_rewrite_terms` | 46 | 6-stage term-pipeline orchestrator. | `query_rewrite.c:476-483` |
| `qo_get_name_by_spec_id` | 219 | Walker: sets `info->appears = true` if any `PT_NAME` matches `info->id`. | `query_rewrite_select.c:620, 3435` |
| `qo_check_nullable_expr` | 241 | Counts expressions whose result *can* be non-NULL despite NULL input. | `optimizer/query_planner.c:11037`, `parser/parser_support.c:3773` (only file in this rewriter exposed beyond optimizer/parser dirs) |
| `qo_check_condition_null` | 315 | Probe: does substituting NULL for every name belonging to `path_spec` make WHERE FALSE? | `query_rewrite_select.c:624` |
| `qo_check_nullable_expr_with_spec` | 357 | Same as `qo_check_nullable_expr` but scoped to a spec. | `query_rewrite_select.c:3434` |
| `qo_is_reduceable_const` | 418 | Treat `PT_CAST` / `PT_TO_ENUMERATION_VALUE` chains around a `PT_IS_CONST_INPUT_HOSTVAR` leaf as constant-equivalent. | `query_rewrite_select.c:2447, 2448, 3026, 3027` |
| `qo_reduce_equality_terms` | 448 | Equality propagation + transitive-join inference. | header-exposed via `QO_CHECK_AND_REDUCE_EQUALITY_TERMS` macro. |
| `qo_reduce_equality_terms_post` | 987 | Walks SELECT subtree, invokes `qo_reduce_equality_terms` per select where (post-order). | `query_rewrite.c:442` |

## Data-shape primer

### CNF × DNF layout

```
WHERE clause: -+- term1 ---or_next--- alt1 ---or_next--- alt2
              |
              +- term2 ---or_next--- alt1
              |
              +- term3 (singleton)
```

- Outer linked-list = CNF (`->next`).
- Inner linked-list = DNF (`->or_next`).

Many helpers explicitly bail when `or_next != NULL` — they assume single-term CNFs (e.g. `qo_apply_range_intersection` only intersects single-term CNF; comment at :4050 cites "implementation complexity").

### `PT_BETWEEN` sub-ops

| Sub-op | Meaning | arg1 | arg2 |
|---|---|---|---|
| `PT_BETWEEN_GE_LE` | `lo ≤ x ≤ hi` | lo | hi |
| `PT_BETWEEN_GE_LT` | `lo ≤ x < hi` | lo | hi |
| `PT_BETWEEN_GT_LE` | `lo < x ≤ hi` | lo | hi |
| `PT_BETWEEN_GT_LT` | `lo < x < hi` | lo | hi |
| `PT_BETWEEN_EQ_NA` | `x = lo` (singleton) | lo | NULL (transient duplicate during merging, nulled in post-pass) |
| `PT_BETWEEN_INF_LE` | `x ≤ hi` | −∞ | hi |
| `PT_BETWEEN_INF_LT` | `x < hi` | −∞ | hi |
| `PT_BETWEEN_GE_INF` | `x ≥ lo` | lo | +∞ |
| `PT_BETWEEN_GT_INF` | `x > lo` | lo | +∞ |

### `COMP_DBVALUE_WITH_OPTYPE_RESULT`

```c
enum comp_dbvalue_with_optye_result {
  CompResultLess       = -2,  /* less than */
  CompResultLessAdj    = -1,  /* less than and adjacent (boundary value equal, strictness differs) */
  CompResultEqual      =  0,
  CompResultGreaterAdj =  1,
  CompResultGreater    =  2,
  CompResultError      =  3
};
```

The `Adj` ("adjacent") values represent the case where the boundary values are equal but strictness differs: `a > 5` vs `a >= 5` at value 5. Critical for adjacency-aware merging — they're what lets `(a > 5) OR (a = 5)` collapse to `(a >= 5)`.

### `DNF_MERGE_RANGE_RESULT`

```c
enum dnf_merge_range_result { DNF_RANGE_VALID = 0, DNF_RANGE_ALWAYS_FALSE = 1, DNF_RANGE_ALWAYS_TRUE = 2 };
```

Returned by `qo_merge_range_helper`. Caller maps `_ALWAYS_FALSE` to `0 PT_NE 0` (strictly-false term) and `_ALWAYS_TRUE` to `IS_NOT_NULL`.

### `PT_EXPR` flags this file mutates

- `PT_EXPR_INFO_TRANSITIVE` — set on cloned join terms inferred by `qo_reduce_equality_terms` (line 920). Planner uses this for cardinality estimation.
- `PT_EXPR_INFO_DO_NOT_AUTOPARAM` — set at line 609 on cloned derived-table-substituted equality terms so the same constant doesn't get re-parameterized after constant propagation.
- `PT_EXPR_INFO_EMPTY_RANGE` — set transiently during `qo_apply_range_intersection_helper` (line 3941) for batched deletion at end (3989-4010).

## Major transformations

### Equality reduction & transitive-join inference (`qo_reduce_equality_terms`, :448)

Multi-phase. Per-CNF-term:

1. Skip if `or_next != NULL` (only single-term DNFs reducible).
2. Accept either `attr = const` (`PT_EQ`) or `attr RANGE(const =)` (`PT_BETWEEN_EQ_NA` inside a one-spec range).
3. Identify which side is the attribute: supports `PT_PRIOR` peeling (528-532), `qo_is_cast_attr` for `CAST(attr)` LHS (545-548), and **derived-table column flattening** (578-622) — if the attribute references a `PT_IS_SUBQUERY` derived spec whose corresponding select-list column is a `PT_NAME_INFO_CONSTANT` or itself reducible, propagate inward; the working term is also cloned with `PT_EXPR_INFO_DO_NOT_AUTOPARAM` (line 609).
4. **Type adaptation for parameterized types** (826-887): when LHS is parameterized (CHAR(n), NUMERIC(p,s), ENUM, …), the constant is cast/wrapped via `tp_value_cast_force` or `pt_wrap_with_cast_op`. Literal precision over `DB_MAX_LITERAL_PRECISION` (255, defined in `query_rewrite.h:39`) takes the wrap path.
5. **Substitution** (889-900): `pt_lambda_with_arg` is the workhorse, with `dont_replace = true` when substituting in the SELECT list (so SELECT keeps the column name) and `false` for the WHERE. **Outer-join awareness**: `temp->info.name.location > 0 ? true : false` is passed so substitution skips terms across outer-join location boundaries.
6. **Transitive-join inference** (666-925): for every other CNF term that is symmetric (`pt_is_symmetric_op`), `qo_collect_name_spec` walks both arguments recording how many times the reduced attribute name appears (`c_name_num`) and which other specs participate (`s_point_list`). Four CASE branches (720-794) qualify a term as a join-term, which is then **cloned**, marked `PT_EXPR_INFO_TRANSITIVE` (920), and appended back to WHERE. PRIOR semantics: `PRIOR field = exp1 AND PRIOR field = exp2` → `PRIOR field = exp1 AND exp1 = exp2` (comment at 442-446).
7. **Trivial `v=v` cut-off** (927-973): final pass nukes `PT_EQ(v,v)` survivors when `db_value_compare(dbv1, dbv2) == DB_EQ`.

The macro `QO_CHECK_AND_REDUCE_EQUALITY_TERMS` (`query_rewrite.h:153-160`) is the idempotency latch on `node->flag.done_reduce_equality_terms` — guards against double-reduction.

### Sargable-form conversion (`qo_converse_sarg_terms`, :1035)

Two-pass per CNF term:

**Pass 1** (1049-1174) — build `attr_list` of attributes appearing in this CNF term, counting frequency in `attr->line_number` (yes, the parser's source-line field is **abused as a counter**; this works because the attr nodes used are `pt_point` aliases). Special UNARY_MINUS handling: `-attr BETWEEN x AND y` is split into `-attr >= x AND -attr <= y` (1087-1106) **only when** there is exactly one CNF and one DNF spec — otherwise correctness across OR-chains breaks.

**Pass 2** (1176-1427) — apply `pt_converse_op` to flip `op_type` so attribute always lands on LHS. Sub-cases:

| Pattern | Rewrite |
|---|---|
| `-x op -y` | `x op y` (preserving PRIOR wrappers) |
| `-attr op const` | `attr op -const` (UNARY_MINUS node reused as wrapper for new constant) |
| `const op -attr` | symmetric |
| `attr op attr` | swap so the more-frequently-appearing attribute is on LHS |
| `non-attr op attr` | straight swap |

The `attr op attr` heuristic picks the side with more references in the rest of the predicate so subsequent range merging hits more pairs.

### Pair-merge → BETWEEN (`qo_reduce_comp_pair_terms`, :1702)

Translates `a >= 10 AND a <= 20` → `a BETWEEN 10 GE_LE 20`. Helper `qo_search_comp_pair_term` (1602) finds a CNF sibling with the *opposite-direction* op (`PT_GE/GT` ↔ `PT_LE/LT`) and matching LHS.

- **PRIOR locking** (1748-1753) — both sides must equally have or lack PRIOR.
- `pt_comp_to_between_op(..., PT_REDUCE_COMP_PAIR_TERMS, ...)` derives the proper `PT_BETWEEN_*` sub-op.
- **Validity check** (1790-1872): if both bounds are constants and `lower > upper` (or `lower == upper` with strict inequality), the conjunct is empty → either:
  - whole WHERE replaced with `PT_VALUE 0` (location 0 case, 1804-1823), or
  - same-`location` (outer-join ON) cluster wiped and `0` value appended (1824-1868).

### LIKE simplification (`qo_rewrite_like_terms`, :2456)

The most isolated cluster. Decision tree per LIKE term, based on `db_get_info_for_like_optimization`:

| Pattern shape | Rewrite |
|---|---|
| No wildcards, no escaping | `PT_EQ` |
| Single `%` | `PT_IS_NOT_NULL` |
| Trailing `%` only AND collation allows like-rewrite | `PT_BETWEEN GE_LT (lower, upper)` |
| Otherwise | "generic": insert synthetic `BETWEEN(LIKE_LOWER_BOUND, LIKE_UPPER_BOUND)` *alongside* the original `LIKE` — keeps original for residual filtering, prepends a runtime-evaluated index-scannable BETWEEN |

Collation gate: `lang_get_collation(coll)->options.allow_like_rewrite` (line 2144). Per-collation switch — some collations (binary, locale-specific) prohibit the rewrite because `LIKE` semantics differ from `BETWEEN` semantics.

`PT_LIKE_LOWER_BOUND` / `PT_LIKE_UPPER_BOUND` are synthetic operators evaluated at scan time; `qo_allocate_like_bound_for_index_scan` (2206) wraps with these and honors `PRM_ID_REQUIRE_LIKE_ESCAPE_CHARACTER` and `PRM_ID_NO_BACKSLASH_ESCAPES`.

`qo_check_like_expression_pre` (2424) excludes `PT_QUERY`, `PT_DOT_`, and uncorrelated `PT_NAME` patterns — these are not pseudo-constant for the index-scan path even when `pt_is_pseudo_const` says yes.

> [!warning] PT_NOT_LIKE not optimized
> TODO at :2487 — `PT_NOT_LIKE` is not optimized at all in this file.

### `PT_RANGE` construction (`qo_convert_to_range`, :3526)

Operator → BETWEEN sub-op:

| Operator | `PT_BETWEEN_*` sub-op |
|---|---|
| `PT_EQ` | `EQ_NA` (`arg1=value, arg2=NULL`) |
| `PT_GT` | `GT_INF` |
| `PT_GE` | `GE_INF` |
| `PT_LT` | `INF_LT` |
| `PT_LE` | `INF_LE` |
| `PT_BETWEEN` w/ `PT_BETWEEN_AND` | `GE_LE` |
| `PT_IS_IN` | list of `EQ_NA` (via `qo_set_value_to_range_list`, :2605) |
| `PT_RANGE` | already converted; no-op |

After conversion, walks DNF (`or_next`) siblings on the same attribute (with PRIOR-pairing, function-index, instnum awareness) and **fuses** them into one `PT_RANGE` node.

Special early-exits:
- A *single* `PT_EQ` term left untouched at :3588 (don't wrap a single equality in PT_RANGE — preserves max parameterizability for downstream auto-parameterize).
- `IN (...)` with a set arg2 and no `or_next` does NOT subsequently call `qo_merge_range_helper` (:3608) — merging huge IN lists is expensive; server eliminates duplicate keys downstream.

### Range merging (DNF-OR-union) — `qo_merge_range_helper`, :3103

Nested loop over `or_next` DNF list. Uses `qo_compare_dbvalue_with_optype` (:2967) for boundary comparison (with ±∞ handling) and `qo_range_optype_rank` (:3074) for tie-breaking: `EQ < {GT, LT} < {GE, LE} < {GT_INF, LT_INF}`.

Steps per pair:
1. Skip non-constant arg1/arg2.
2. Disjoint-range detection (3272-3277).
3. Empty-range detection (3263-3273) — guards against `a > 1 OR a between 1 and 0`.
4. Lower-bound merge (3294-3355): if sibling extends lower, swap arg1 over; rank-aware tie-break.
5. Upper-bound merge (3358-3406): symmetric.

**Bound-arg swap "trick"** (3282-3291): for `INF_LT` / `INF_LE` the algorithm temporarily moves arg1→arg2 to share the uniform `arg1=lower / arg2=upper` algorithms; reverted at 3436-3439.

**EQ_NA cleanup** (3501-3508): post-pass nulls out `arg2` of any `PT_BETWEEN_EQ_NA` because the algorithm may have populated it transiently.

Returns `DNF_RANGE_VALID` / `_ALWAYS_FALSE` / `_ALWAYS_TRUE`. Caller (`qo_convert_to_range:3629-3648`) maps `_ALWAYS_FALSE` → `0 PT_NE 0` and `_ALWAYS_TRUE` → `IS_NOT_NULL`.

### Range intersection (CNF-AND) — `qo_apply_range_intersection`, :4038

Handles `WHERE a RANGE(...) AND a RANGE(...)`:

- Per-spec walker over node1's DNF; for each, scans node2's DNF for non-disjoint siblings.
- Empty-range intersections marked `PT_EXPR_INFO_EMPTY_RANGE` for batched deletion at end (3989-4010) — only if `dont_remove_sibling` is false (caller's safety latch when at least one operand was non-constant).
- `pt_comp_to_between_op(..., PT_RANGE_INTERSECTION, ...)` builds the merged sub-op.

**Restriction**: only single-term CNF with no `or_next` is intersected (:4048, 4050). Self-contradictory ranges wipe `arg2 = NULL`, triggering empty-conjunction collapse (4198-4288, location-aware false-collapse mirroring `qo_reduce_comp_pair_terms`).

### IS NULL / IS NOT NULL folding (`qo_fold_is_and_not_null`, :1445)

**Sibling-driven** (1473-1537): if any other CNF term at the same `expr.location` references the same attribute (modulo `pt_check_path_eq` + PRIOR-matching at 1490-1495):

| Sibling | Result for this `IS NULL` | Result for this `IS NOT NULL` |
|---|---|---|
| `IS NULL` | TRUE (idempotent) | FALSE |
| `IS NOT NULL` | FALSE | TRUE |
| `<=> NULL` (NULLSAFE-EQ to NULL) | TRUE | FALSE |
| `<=> non-NULL` | bail out (TODO) |
| Other comparison (e.g. `a < 10`) | FALSE (because `a < 10` already implies `a IS NOT NULL`) | TRUE |

**Schema-driven** (1538-1565): a `PT_IS_NOT_NULL` on a column with NOT-NULL constraint **and not on the nullable side of an outer join** (`mq_is_outer_join_spec`, 1555) folds to TRUE. Outer-join discipline is the load-bearing safety here — without it, this would over-eliminate in left-outer-joined nullable spec scenarios.

The folded `PT_VALUE` inherits `node->info.expr.location`, so location-scoped collapse downstream still sees consistent data.

## Hot / tricky invariants

1. **CNF shape assumed.** Every public function (except recursive descents in `qo_converse_sarg_terms` and the LIKE walkers) assumes `*wherep` is already CNF. AND/OR survival inside a DNF triggers either `continue` or recursive descent.
2. **PRIOR pairing.** A LHS-with-PRIOR may only be merged/intersected/folded with a RHS-with-PRIOR. Checked at 1748-1753 (pair-merge), 2800-2817 (range-convert sibling), 4118-4135 (range-intersect sibling), 1490-1495 (NULL-fold). Missing this would alias inner-row vs outer-row attributes in CONNECT BY traversal.
3. **`location` discipline.** Every cross-term comparison gates on `expr.location` equality (1483, 1650, 4146); any falsification respects location-scope (1804/4216). This is the core outer-join correctness invariant — terms from different ON clauses must not interact.
4. **EQ_NA arg2 nulling.** Multiple post-loops (3501-3508, 4013-4028) re-null arg2 of `PT_BETWEEN_EQ_NA` because intermediate steps temporarily duplicate arg1→arg2 to share algorithms with two-arg sub-ops.
5. **`Adj` adjacency semantics.** `LessAdj`/`GreaterAdj` mean "boundary equal but operator strictness differs". Crucial for adjacency-aware union/intersection.
6. **Idempotency.** `qo_reduce_equality_terms` is idempotency-latched via `node->flag.done_reduce_equality_terms`. The other six stages assume single invocation per phase. Calling twice is mostly harmless; `qo_convert_to_range`'s `case PT_RANGE: return;` (:2767) is the only explicit re-entry guard.
7. **Three-valued logic.** Folding to `PT_VALUE 0` (false) only happens when an *empty range* is *provable from constants* — never from variable comparisons. NULL-fold respects 3VL by using `IS NOT NULL`/`IS NULL` semantics directly rather than a generic `= NULL → FALSE` rewrite.
8. **Auto-param safety.** `PT_EXPR_INFO_DO_NOT_AUTOPARAM` set on cloned equality terms (:609); single-`PT_EQ` refusal in `qo_convert_to_range` (:3588) is partly motivated by host-var parameterizability downstream.

## Internal data structures

All declared in `query_rewrite.h`:

- `enum comp_dbvalue_with_optye_result` (typo: should be `optype`).
- `enum dnf_merge_range_result`.
- `struct spec_id_info {UINTPTR id; bool appears, nullable}` — used by `qo_get_name_by_spec_id`.
- `struct pt_name_spec_info {c_name, c_name_num, query_serial_num, s_point_list}` — gathers spec-IDs around a candidate reduced-attr name during transitive-join inference.
- Macros: `QO_CHECK_AND_REDUCE_EQUALITY_TERMS`, `PROCESS_IF_EXISTS`.
- `#define DB_MAX_LITERAL_PRECISION 255` — guard for parameterized-type cast vs wrap.

`attr->line_number` is **abused** as an attribute-appearance frequency counter in `qo_converse_sarg_terms` (1131, 1142, 1155, 1167) — works because the attr nodes used are `pt_point` aliases, but obscure.

No file-scope statics, no module-private lookup tables.

## Static helper one-liners

| Helper | Line | Purpose |
|---|---:|---|
| `qo_collect_name_spec` | 69 | Pre-walk: tally `info->c_name` appearances; collect other specs into `s_point_list`; give up on subquery/serial. |
| `qo_collect_name_spec_post` | 196 | Post-walk: stop if subquery seen. |
| `qo_replace_spec_name_null` | 281 | Substitute `PT_TYPE_NULL` for every name belonging to one spec. |
| `qo_is_cast_attr` | 399 | True iff the expression is `PT_CAST(attr)`. |
| `qo_search_comp_pair_term` | 1602 | Find opposite-direction comparison sibling for BETWEEN-pair-merge. |
| `qo_find_like_rewrite_bound` | 1893 | Synthesize lower or upper bound DB_VALUE for a LIKE rewrite. |
| `qo_rewrite_one_like_term` | 1979 | Per-LIKE decision tree. |
| `qo_allocate_like_bound_for_index_scan` | 2206 | Wrap pattern with `PT_LIKE_LOWER_BOUND` / `PT_LIKE_UPPER_BOUND`. |
| `qo_rewrite_like_for_index_scan` | 2296 | Insert synthetic BETWEEN alongside original LIKE for index eligibility. |
| `qo_check_like_expression_pre` | 2424 | Exclude `PT_QUERY`, `PT_DOT_`, uncorrelated `PT_NAME` patterns. |
| `qo_set_value_to_range_list` | 2605 | `PT_VALUE` / `PT_FUNCTION` / `PT_NAME` → linked list of `PT_BETWEEN_EQ_NA`. |
| `qo_convert_to_range_helper` | 2669 | Turn one comparison/BETWEEN/IN into `PT_RANGE`, fusing same-attr DNF siblings. |
| `qo_compare_dbvalue_with_optype` | 2967 | Six-way comparison handling ±∞ markers and `Adj` adjacency. |
| `qo_range_optype_rank` | 3074 | Op canonicalization for tie-breaking. |
| `qo_merge_range_helper` | 3103 | DNF-OR-union of `PT_BETWEEN_*` sub-ranges. |
| `qo_apply_range_intersection_helper` | 3668 | CNF-AND-intersection of two `PT_RANGE` nodes' DNF lists. |

## Bugs / smells / TODOs

- **TODO at 2082-2086** — unescaped no-wildcard LIKE patterns with embedded escapes fall back to "generic" instead of being rewritten directly to `PT_EQ` after escape elimination.
- **TODO at 2102-2106** — trailing-space LIKE patterns: index-ordering inconsistency for trailing spaces and "char-code-1 dummy escape" issue inside `qstr_eval_like`.
- **TODO at 2487** — `PT_NOT_LIKE` not optimized.
- **TODO at 2546-2547** — "We should check that the column is indexed". Currently rewrites unconditionally for every column matching the safety filter, even when there's no usable index.
- **TODO at 2504-2505** — `PT_LIKE`'s shape (`{arg1=col, arg2=PT_LIKE_ESCAPE{arg1=pattern, arg2=escape}}`) is awkward; suggests parser should put escape into `PT_LIKE.arg3`.
- **TODO at 1525-1527** — `(a IS NULL AND a <=> expr)` ⇒ `(a IS NULL AND expr IS NULL)`, `(a IS NOT NULL AND a <=> expr)` ⇒ `(a = expr)` — left as future work.
- **Comment typo** at line 26 (file is `query_rewrite_term.c` but says `query_rewrite_predicate.c`) — rename artefact.
- **Enum typo** `comp_dbvalue_with_optye_result` (missing 'p') in public header.
- **Magic 255 cap** `DB_MAX_LITERAL_PRECISION` — reasonable but commented nowhere why precisely 255.
- **`assert(false)` in `qo_range_optype_rank` default branch** (3090) — unexpected op crashes debug builds and silently returns 1 in release.
- **`qo_apply_range_intersection`** :4050 comment ("Due to implementation complexity, handle one predicate term only") — explicit reason CNF terms with `or_next` chains skip intersection.
- **`qo_search_comp_pair_term`** uses `pt_check_class_eq` only on the right argument (1677) when both sides are attributes — no symmetric check on LHS class equality (deliberate? unclear).
- **`attr->line_number` abuse** as frequency counter in `qo_converse_sarg_terms` — relies on `pt_point` aliasing.

## Cross-references

- **Parser helpers** (`parser/parser.h`): `pt_cnf`, `pt_converse_op`, `pt_comp_to_between_op` (takes `PT_COMP_TO_BETWEEN_OP_CODE_TYPE`: `PT_RANGE_MERGE`, `PT_RANGE_INTERSECTION`, `PT_REDUCE_COMP_PAIR_TERMS`), `pt_between_to_comp_op`, `pt_lambda_with_arg`, `pt_semantic_type`, `pt_find_entity`, `pt_false_search_condition`, `pt_check_path_eq`, `pt_check_class_eq`, `pt_get_first_arg_ignore_prior`, `pt_check_not_null_constraint`, `pt_is_pseudo_const`, `pt_is_attr`, `pt_is_const_not_hostvar`, `pt_is_function_index_expression`, `pt_is_instnum`, `pt_is_symmetric_op`, `pt_name_equal`, `pt_point`, `pt_value_to_db`, `pt_dbval_to_value`, `pt_wrap_with_cast_op`, `pt_node_to_db_domain`.
- **View/MQ**: `mq_is_outer_join_spec` — outer-join nullable-side check.
- **Type system**: `tp_value_cast_force`, `tp_domain_cache`, `tp_value_compare`, `db_value_compare`, `pr_clear_value`, `pr_copy_value`, `pr_free_value`, `db_make_int`, `db_make_null`, `db_string_put_cs_and_collation`, `db_value_clear`, `db_get_string*`, `intl_char_count`.
- **LIKE-runtime helpers** (`query/string_opfunc.h`): `db_compress_like_pattern`, `db_get_info_for_like_optimization`, `db_get_like_optimization_bounds`. `LIKE_WILDCARD_MATCH_MANY = '%'`.
- **Collation**: `lang_get_collation` + `options.allow_like_rewrite` flag (line 2144).
- **System parameters**: `PRM_ID_REQUIRE_LIKE_ESCAPE_CHARACTER`, `PRM_ID_NO_BACKSLASH_ESCAPES`.
- **Error reporting**: `PT_INTERNAL_ERROR`, `PT_ERRORmf2`, `PT_ERRORm`, `er_stack_push/pop`, `er_clear`, `er_errid`, `pt_has_error`, `pt_reset_error`.

XASL flag impact: none directly. The output is still a `PT_NODE` tree; XASL is built later by [[components/optimizer]] / `query_planner.c` / `query_graph.c`. The flags this file sets/clears live on `PT_EXPR.info.expr.flag` and influence later passes (transitive marking → planner cardinality estimation; do-not-autoparam → `qo_auto_parameterize`; empty-range → deletion sweep).

## Related

- Parent: [[components/optimizer-rewriter]]
- Sibling: [[components/optimizer-rewriter-select]] (consumes `qo_get_name_by_spec_id`, `qo_check_condition_null`, `qo_check_nullable_expr_with_spec`), [[components/optimizer-rewriter-auto-parameterize]] (consumes the `PT_EXPR_INFO_DO_NOT_AUTOPARAM` flag this file sets).
- [[components/parse-tree]] — `PT_BETWEEN_*`, `PT_RANGE`, `PT_LIKE`, `PT_LIKE_LOWER_BOUND`, `PT_LIKE_UPPER_BOUND` definitions; flag bit assignments.
- [[Query Processing Pipeline]]
