---
type: component
parent_module: "[[modules/src|src]]"
path: "src/xasl/xasl_predicate.hpp"
status: active
purpose: "Predicate expression tree (PRED_EXPR) used in XASL WHERE/HAVING/HAVING/join conditions — Boolean tree of AND/OR/NOT with four eval-term leaves: comparison, ALL-SOME, LIKE, RLIKE"
key_files:
  - "src/xasl/xasl_predicate.hpp (PRED_EXPR, PRED, EVAL_TERM, COMP_EVAL_TERM, ALSM_EVAL_TERM, LIKE_EVAL_TERM, RLIKE_EVAL_TERM)"
tags:
  - component
  - cubrid
  - xasl
  - predicate
related:
  - "[[components/xasl|xasl]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[components/query-executor|query-executor]]"
created: 2026-04-23
updated: 2026-04-23
---

# `PRED_EXPR` — Predicate Expression Tree

`PRED_EXPR` (`cubxasl::pred_expr`) is the boolean predicate tree used throughout `XASL_NODE` for `WHERE`, `HAVING`, join conditions, and key-range filters. It appears in `ACCESS_SPEC_TYPE.where_key`, `ACCESS_SPEC_TYPE.where_pred`, `XASL_NODE.during_join_pred`, `XASL_NODE.after_join_pred`, `XASL_NODE.instnum_pred`, and many others.

## Structure

```
PRED_EXPR  (TYPE_PRED_EXPR discriminant)
├── T_PRED        → pred { lhs: PRED_EXPR*, rhs: PRED_EXPR*, bool_op }
├── T_EVAL_TERM   → eval_term { et_type, union { et_comp / et_alsm / et_like / et_rlike } }
└── T_NOT_TERM    → PRED_EXPR*  (negation)
```

### `pred` (boolean combinator)

```cpp
struct pred {
  pred_expr *lhs;
  pred_expr *rhs;
  BOOL_OP bool_op;   // B_AND, B_OR, B_XOR, B_IS, B_IS_NOT
};
```

### `eval_term` leaves

| `et_type` | Struct | What it evaluates |
|-----------|--------|-------------------|
| `T_COMP_EVAL_TERM` | `comp_eval_term` | Simple comparison: `lhs rel_op rhs` |
| `T_ALSM_EVAL_TERM` | `alsm_eval_term` | `elem rel_op ALL/SOME (elemset)` |
| `T_LIKE_EVAL_TERM` | `like_eval_term` | `src LIKE pattern [ESCAPE esc_char]` |
| `T_RLIKE_EVAL_TERM` | `rlike_eval_term` | `src REGEXP pattern [case_sensitive]` |

### `comp_eval_term` (most common)

```cpp
struct comp_eval_term {
  regu_variable_node *lhs;
  regu_variable_node *rhs;
  REL_OP rel_op;   // R_EQ, R_NE, R_GT, R_GE, R_LT, R_LE, R_NULL, R_EXISTS,
                   // R_LIKE, R_EQ_SOME, R_NE_SOME … R_LE_ALL,
                   // R_SUBSET, R_SUPERSET, R_SUBSETEQ, R_SUPERSETEQ,
                   // R_EQ_TORDER, R_NULLSAFE_EQ
  DB_TYPE type;    // comparison type hint
};
```

### `rlike_eval_term` (note mutable cache)

```cpp
struct rlike_eval_term {
  regu_variable_node *src;
  regu_variable_node *pattern;
  regu_variable_node *case_sensitive;
  mutable cub_compiled_regex *compiled_regex;  // cached at first evaluation
};
```

The `compiled_regex` is a server-side runtime cache — not serialised. It is rebuilt on first RLIKE evaluation from the `pattern` regu variable.

## `REL_OP` enum summary

Standard: `R_EQ R_NE R_GT R_GE R_LT R_LE R_NULL R_EXISTS R_LIKE`  
Set-quantified: `R_EQ_SOME … R_LE_SOME R_EQ_ALL … R_LE_ALL`  
Set membership: `R_SUBSET R_SUPERSET R_SUBSETEQ R_SUPERSETEQ`  
Special: `R_EQ_TORDER` (total-order equality, NULLs compare equal), `R_NULLSAFE_EQ` (`<=>` operator)

## Namespace and aliases

All types are in `namespace cubxasl`. Legacy C aliases:
```cpp
using PRED_EXPR      = cubxasl::pred_expr;
using PRED           = cubxasl::pred;
using EVAL_TERM      = cubxasl::eval_term;
using COMP_EVAL_TERM = cubxasl::comp_eval_term;
using ALSM_EVAL_TERM = cubxasl::alsm_eval_term;
using LIKE_EVAL_TERM = cubxasl::like_eval_term;
using RLIKE_EVAL_TERM= cubxasl::rlike_eval_term;
```

## Related

- [[components/xasl|xasl]] — hub; `PRED_EXPR` appears in access specs and XASL_NODE directly
- [[components/regu-variable|regu-variable]] — eval-term leaves hold `REGU_VARIABLE *` operands
- [[components/xasl-generation|xasl-generation]] — `pt_to_pred_expr` builds `PRED_EXPR` from `PT_EXPR`
- [[components/query-executor|query-executor]] — `eval_pred` evaluates the predicate tree at runtime
- Source: [[sources/cubrid-src-xasl|cubrid-src-xasl]]
