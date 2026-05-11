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
updated: 2026-05-11
---

# Semantic Check & Type Checking

Two passes that run after [[components/name-resolution|name resolution]] and before [[components/xasl-generation|XASL generation]].

- `semantic_check.c` — structural and semantic validation.
- `type_checking.c` — expression type inference; populates `PT_NODE.type_enum` on every expression node.

Both are called from `pt_compile` (in `compile.c`) and share the `SEMANTIC_CHK_INFO` struct.

> [!update] 2026-05-11 — `pt_check_with_info` 4-단계 (per [[sources/qp-analysis-semantic-check]])
> 본 모듈의 외부 entry `pt_semantic_check` 가 내부적으로 `pt_check_with_info` 를 호출하며, `PT_SELECT` 의 경우 다음 4 단계 (+ 1 path-expansion) 가 정확한 순서로 실행됨:
>
> 1. **`pt_resolve_names()`** — name resolution ([[components/name-resolution]] 위임). 5단계 sub-walk: `pt_flat_spec_pre` → `pt_bind_names`/`pt_bind_names_post` → `pt_resolve_group_having_alias` → `pt_resolve_natural_join` → `SELECT … FOR UPDATE` flag 설정.
> 2. **`pt_check_where()`** — `WHERE` 절에 aggregate/analytic 함수 금지. 발견 시 `MSGCAT_SEMANTIC_INVALID_AGGREGATE` / `MSGCAT_SEMANTIC_NESTED_ANALYTIC_FUNCTIONS`.
> 3. **`pt_check_and_replace_hostvar()`** — `?` (`PT_HOST_VAR`) → `PT_VALUE` 변환.
> 4. **`pt_semantic_check_local()`** — statement-별 체크 (illegal multi-column subquery, INTO 갯수 검사, WITH INCREMENT rewrite, aggregate/analytic 보유시 GROUP BY/HAVING 검사, ORDER BY 정리, CONNECT BY, SHOW 메타rewrite, derived query ambiguous reference, CAST well-formed, **LIMIT → Numbering Expression rewrite** (ORDERBY_NUM/GROUPBY_NUM/INST_NUM), CNF 의 search condition cut-off + fold-as-false). 마지막에 `pt_semantic_type()` 호출.
> 5. **`pt_expand_isnull_preds()`** — path expression 관련 (분석 생략).
>
> ### type_checking 의 6 단계 (`pt_eval_expr_type()`)
> 1. **Rule-based** — `PT_PLUS/PT_MINUS/PT_BETWEEN*/PT_LIKE*/PT_IS_IN*/PT_TO_CHAR/PT_FROM_TZ/PT_NEW_TIME` 등 op 별 implicit conversion 룰.
> 2. **Expression definition based** — `pt_apply_expressions_definition()` → `pt_get_expression_definition()` 의 explicit conversion table 에서 best_match + `pt_coerce_expr_arguments()`.
> 3. **Collation compatible** — `pt_check_expr_collation()`.
> 4. **Host variable late binding** — `pt_is_op_hv_late_bind()` 가 후보 op 판정. `expected_domain` CAST 삽입 + XASL gen 단계에서 `DB_TYPE_VARIABLE` domain 처리하기 위해 `expected_domain = NULL`.
> 5. **자명한 결과 타입** — `PT_BETWEEN`, `PT_LIKE`, `PT_RAND/RANDOM/DRAND/DRANDOM`, `PT_EXTRACT`, `PT_YEAR/MONTH/DAY`, `PT_HOUR/MINUTE/SECOND`, `PT_MILLISECOND`, `PT_COALESCE`, `PT_FROM_TZ`.
> 6. **타입 정해진 host variable 인수에 expected domain 설정**.
>
> ### Recursive Expression (`pt_eval_type_pre()`)
> `GREATEST/LEAST/COALESCE` (Left-Recursive) 와 `CASE/DECODE` (Right-Recursive) — 같은 operator 가 트리 왼/오른쪽으로 연쇄되는 경우 **한 expression 으로 묶어 일괄 타입 결정**. `eval_recursive_expr_type()` 가 맨 하단까지 재귀 하강 후 `eval_expr_type()` 으로 펼치며 올라옴.
>
> ### Function type evaluation
> C 클래스 도입 (`func_type` namespace 의 `Node` 클래스). `func_all_signatures` 와 매칭하여 인수/반환 타입 지정 — `pt_eval_function_type_new()`. legacy 경로 `pt_eval_function_type_old()` 와 분기.
>
> ### Constant Folding 의 escape hatch
> `pt_fold_constants_pre()` 가 `benchmark()` 함수 호출이 있으면 PT_LIST_WALK 로 자식 진입 차단 → CAS 에서 fold 되지 않고 executor 까지 가도록.

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
- Source (internal R&D wiki): [[sources/qp-analysis-semantic-check]] — 4 PDFs (Overview, Name Resolution v0.9 22p, Type Check & Constant Fold, Particular Statement)
