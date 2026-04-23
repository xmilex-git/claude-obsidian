---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/regu_var.hpp"
status: active
purpose: "REGU_VARIABLE (regu_variable_node) — the universal expression atom in XASL; encodes column references, constants, arithmetic, function calls, aggregate links, subquery pointers, and stored-procedure calls as a deeply nested discriminated union"
key_files:
  - "src/query/regu_var.hpp (regu_variable_node class, REGU_DATATYPE enum, ARITH_TYPE, FUNCTION_TYPE, ATTR_DESCR)"
  - "src/parser/xasl_generation.c (pt_to_regu_variable — PT_EXPR → REGU_VARIABLE translation)"
  - "src/query/query_executor.c (fetch_val_list, qexec_eval_regu — evaluation)"
  - "src/query/xasl_to_stream.c (xts_save_regu_variable)"
  - "src/query/stream_to_xasl.c (stx_build — regu_variable_node overload)"
tags:
  - component
  - cubrid
  - xasl
  - regu-variable
  - query
related:
  - "[[components/xasl|xasl]]"
  - "[[components/xasl-predicate|xasl-predicate]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/xasl-stream|xasl-stream]]"
created: 2026-04-23
updated: 2026-04-23
---

# `REGU_VARIABLE` — Register Variable

`REGU_VARIABLE` (typedef for `regu_variable_node`) is the **expression atom** of the XASL execution model. Every column reference, constant, arithmetic expression, function call, aggregate result reference, and subquery link in an `XASL_NODE` plan is ultimately a `REGU_VARIABLE`.

> [!key-insight] Single universal expression type
> CUBRID uses one class (`regu_variable_node`) for all expression nodes. The `type` field (`REGU_DATATYPE` enum) discriminates which member of the `value` union is valid. This makes generic walkers (`map_regu`) possible but also makes the type heavily overloaded — the AGENTS.md calls it "the most complex type — deeply nested unions".

## `REGU_DATATYPE` — 17-way discriminant

Full enum with value semantics. The `value` union member listed is valid when `type == …`:

| Constant | `value` member | Evaluation time | Description |
|----------|---------------|-----------------|-------------|
| `TYPE_DBVAL` | `dbval` (DB_VALUE inline) | Compile time | Literal constant — embedded in the XASL stream |
| `TYPE_CONSTANT` | `dbvalptr` (DB_VALUE *) | Prepare time | Points to host-var slot or cached bound value |
| `TYPE_ORDERBY_NUM` | `dbvalptr` (DB_VALUE *) | Per tuple | `ORDERBY_NUM()` result slot |
| `TYPE_INARITH` | `arithptr` (ARITH_TYPE *) | Per tuple (once) | Input arithmetic — evaluated once, result cached in `arithptr->value` |
| `TYPE_OUTARITH` | `arithptr` (ARITH_TYPE *) | Per tuple (always) | Output arithmetic — always re-evaluated |
| `TYPE_ATTR_ID` | `attr_descr` (ATTR_DESCR) | Per tuple | Fetch heap attribute from current tuple |
| `TYPE_CLASS_ATTR_ID` | `attr_descr` (ATTR_DESCR) | Per access | Class-level (static) attribute |
| `TYPE_SHARED_ATTR_ID` | `attr_descr` (ATTR_DESCR) | Per access | Shared attribute |
| `TYPE_POSITION` | `pos_descr` (QFILE_TUPLE_VALUE_POSITION) | Per tuple | Positional reference into list-file tuple columns |
| `TYPE_LIST_ID` | `srlist_id` (QFILE_SORTED_LIST_ID *) | Per subquery execution | Sorted list file (subquery result, used for IN / EXISTS) |
| `TYPE_POS_VALUE` | `val_pos` (int) | Per binding | Positional host variable (CAS binding slot index) |
| `TYPE_OID` | — (no union field) | Per tuple | Current scan object OID |
| `TYPE_CLASSOID` | — (no union field) | Per tuple | Current scan class OID |
| `TYPE_FUNC` | `funcp` (FUNCTION_TYPE *) | Per tuple | Built-in scalar function call (regex, JSON, string, etc.) |
| `TYPE_REGUVAL_LIST` | `reguval_list` (REGU_VALUE_LIST *) | Per row | VALUES list for multi-row INSERT |
| `TYPE_REGU_VAR_LIST` | `regu_var_list` (REGU_VARIABLE_LIST) | Per partition | `CUME_DIST` / `PERCENT_RANK` argument list |
| `TYPE_SP` | `sp_ptr` (sp_node *) | Per tuple | Stored procedure invocation |

## Class layout

```cpp
class regu_variable_node {
public:
  REGU_DATATYPE   type;             // discriminant
  int             flags;            // REGU_VARIABLE_* bitmask
  TP_DOMAIN      *domain;           // result domain
  TP_DOMAIN      *original_domain;  // pre-clone domain
  DB_VALUE       *vfetch_to;        // fetch-target for qp_fetchvlist
  xasl_node      *xasl;            // linked subquery XASL (if any)
  union regu_data_value {
    DB_VALUE              dbval;        // TYPE_DBVAL
    DB_VALUE             *dbvalptr;     // TYPE_CONSTANT / TYPE_ORDERBY_NUM
    ARITH_TYPE           *arithptr;     // TYPE_INARITH / TYPE_OUTARITH
    aggregate_list_node  *aggptr;       // aggregate link (server-only path)
    ATTR_DESCR            attr_descr;   // TYPE_ATTR_ID etc.
    QFILE_TUPLE_VALUE_POSITION pos_descr; // TYPE_POSITION
    QFILE_SORTED_LIST_ID *srlist_id;    // TYPE_LIST_ID
    int                   val_pos;      // TYPE_POS_VALUE
    FUNCTION_TYPE        *funcp;        // TYPE_FUNC
    REGU_VALUE_LIST      *reguval_list; // TYPE_REGUVAL_LIST
    REGU_VARIABLE_LIST    regu_var_list;// TYPE_REGU_VAR_LIST
    sp_node              *sp_ptr;       // TYPE_SP
  } value;
};
```

## Key flags

| Flag constant | Hex | Meaning |
|---------------|-----|---------|
| `REGU_VARIABLE_HIDDEN_COLUMN` | `0x01` | Does not go to list file output |
| `REGU_VARIABLE_FIELD_COMPARE` | `0x02` | Bottom of `FIELD()` function regu tree |
| `REGU_VARIABLE_FIELD_NESTED` | `0x04` | Child in `FIELD()` tree |
| `REGU_VARIABLE_APPLY_COLLATION` | `0x08` | Apply collation from domain (`COLLATE` modifier) |
| `REGU_VARIABLE_ANALYTIC_WINDOW` | `0x10` | Analytic window function context |
| `REGU_VARIABLE_FETCH_ALL_CONST` | `0x40` | All sub-expressions are constant |
| `REGU_VARIABLE_FETCH_NOT_CONST` | `0x80` | Contains a non-constant |
| `REGU_VARIABLE_CLEAR_AT_CLONE_DECACHE` | `0x100` | Clear at XASL clone decache |
| `REGU_VARIABLE_CORRELATED` | `0x800` | Correlated scalar subquery result cache |

## Arithmetic expressions (`ARITH_TYPE`)

```c
struct arith_list_node {  // typedef'd ARITH_TYPE
  TP_DOMAIN    *domain;
  DB_VALUE     *value;          // result slot
  REGU_VARIABLE *leftptr;       // left operand (recursive)
  REGU_VARIABLE *rightptr;      // right operand (recursive)
  REGU_VARIABLE *thirdptr;      // third operand (e.g. BETWEEN, SUBSTR)
  OPERATOR_TYPE  opcode;        // PT_PLUS, PT_MINUS, PT_CASE, …
  MISC_OPERAND   misc_operand;  // trim qualifier / datetime field
  pred_expr     *pred;          // predicate inside arithmetic (e.g. CASE WHEN)
  struct drand48_data *rand_seed; // server-only; for RAND()
};
```

## Function calls (`FUNCTION_TYPE`)

```c
struct function_node {  // typedef'd FUNCTION_TYPE
  DB_VALUE          *value;    // result slot
  REGU_VARIABLE_LIST operand;  // argument list
  FUNC_CODE          ftype;    // PT_SUBSTRING, PT_CONCAT, PT_JSON_*, …
  function_tmp_obj  *tmp_obj;  // compiled regex or other temp state
};
```

## Attribute descriptor (`ATTR_DESCR`)

Used for `TYPE_ATTR_ID` — fetches an attribute value from the current heap tuple:

```c
struct attr_descr_node {  // typedef'd ATTR_DESCR
  CL_ATTR_ID          id;             // attribute identifier
  DB_TYPE             type;
  HEAP_CACHE_ATTRINFO *cache_attrinfo; // catalog cache
  DB_VALUE            *cache_dbvalp;   // cached value pointer
};
```

## Subquery link (`xasl` field)

When `regu_variable_node::xasl != NULL`, the variable is a subquery reference. The macro `EXECUTE_REGU_VARIABLE_XASL(thread_p, r, v)` in `xasl.h` drives execution:
- Checks `XASL_LINK_TO_REGU_VARIABLE` flag on the subquery node.
- Calls `qexec_execute_mainblock` or `qexec_execute_subquery_for_result_cache` on first evaluation.
- Sets `r->value.dbvalptr` to the single-tuple result value.
- Status transitions: `XASL_INITIALIZED → XASL_SUCCESS / XASL_FAILURE`.

## Per-tuple vs compile-time evaluation

`REGU_VARIABLE` types fall into three temporal categories:

1. **Compile-time / stream constants**: `TYPE_DBVAL` — the `DB_VALUE` is packed directly into the XASL stream; evaluation is just a pointer dereference.
2. **Per-tuple evaluated**: `TYPE_INARITH` (once-per-group), `TYPE_OUTARITH` (always), `TYPE_ATTR_ID`, `TYPE_FUNC`, `TYPE_POSITION` — re-evaluated on every call to `fetch_val_list` / `qexec_eval_regu`.
3. **Subquery-driven**: `TYPE_LIST_ID` — the `QFILE_SORTED_LIST_ID` is populated by executing a subquery `XASL_NODE`; the `EXECUTE_REGU_VARIABLE_XASL` macro drives this path. Result is cached for the subquery's scope (see [[components/subquery-cache|subquery-cache]] for the per-execution memoization layer).

`TYPE_INARITH` vs `TYPE_OUTARITH` distinction: `INARITH` expressions appear in the scan predicate (computed once per group key); `OUTARITH` appear in the SELECT output list (computed once per output tuple). The executor checks `REGU_VARIABLE_FETCH_NOT_CONST` / `REGU_VARIABLE_FETCH_ALL_CONST` flags before evaluating to short-circuit constant sub-expressions.

---

## Recursive walker — actual coverage

```cpp
// Apply func to all REGU_VARIABLEs reachable from this node.
void regu_variable_node::map_regu(const map_regu_func_type &func);

// Also walk into nested XASL subtrees.
void regu_variable_node::map_regu_and_xasl(
  const map_regu_func_type &regu_func,
  const map_xasl_func_type &xasl_func);
```

`map_regu` (in `regu_var.cpp`) visits children for: `TYPE_INARITH`, `TYPE_OUTARITH` (left + right via `arithptr`), `TYPE_SP` (all args), `TYPE_FUNC` (all operands), `TYPE_REGUVAL_LIST` (all items), `TYPE_REGU_VAR_LIST` (all nodes). ARITH `thirdptr` is **not** visited by `map_regu` — only left and right are traversed.

> [!warning] map_regu does not visit ARITH thirdptr
> `ARITH_TYPE.thirdptr` (used by BETWEEN, SUBSTR, etc.) is NOT visited by `map_regu`. Applications that use `map_regu` for tree modification must handle the three-operand case separately or use a direct walk of `ARITH_TYPE`.

`map_regu_and_xasl` wraps `map_regu` with an additional callback for any `XASL_NODE *` pointers (via `regu.xasl`) found during the walk. This is used for clone-decache operations that must clear sub-plan accumulators.

## Serialisation

`regu_variable_node` has a dedicated `stx_build` overload (in `xasl_stream.hpp`). The entire union is packed field-by-field; the `xasl` pointer becomes a stream offset resolved by `stx_restore<xasl_node>`.

The `flags`, `domain`, and `original_domain` are serialised; `vfetch_to` and the server-only parts of sub-structures are initialised at execution time.

## Related

- [[components/xasl|xasl]] — hub: XASL_NODE and how regu variables are used
- [[components/xasl-predicate|xasl-predicate]] — `PRED_EXPR` whose eval_terms contain `REGU_VARIABLE *`
- [[components/xasl-generation|xasl-generation]] — `pt_to_regu_variable` creates REGU_VARIABLEs from PT_EXPRs
- [[components/query-executor|query-executor]] — evaluates regu variables at runtime
- [[components/xasl-stream|xasl-stream]] — serialisation protocol
- Source: [[sources/cubrid-src-xasl|cubrid-src-xasl]]
