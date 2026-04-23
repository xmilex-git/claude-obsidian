---
type: component
title: "query-fetch — REGU_VARIABLE Evaluation and Tuple Value Fetch"
parent_module: "[[modules/src|src]]"
path: src/query/fetch.{c,h}
status: developing
key_files:
  - src/query/fetch.c
  - src/query/fetch.h
public_api:
  - fetch_peek_dbval
  - fetch_copy_dbval
  - fetch_val_list
  - fetch_init_val_list
  - fetch_force_not_const_recursive
tags:
  - cubrid
  - query
  - fetch
  - regu-variable
  - expression-evaluation
  - server-side
---

# query-fetch — REGU_VARIABLE Evaluation and Tuple Value Fetch

> [!key-insight]
> Despite the name, `fetch.c` is **not** the cursor/result-set pagination layer. It is the **expression evaluator** for `REGU_VARIABLE` nodes at scan time. Every predicate check and projection column evaluation inside the query executor calls `fetch_peek_dbval` or `fetch_copy_dbval`. "Fetch" here means "fetch the DB_VALUE for this expression atom from the current tuple context."

## Purpose

`fetch.c` (~5 K lines) evaluates `REGU_VARIABLE` expressions against a tuple context:

- **Peek** (`fetch_peek_dbval`): returns a pointer to an existing `DB_VALUE` already materialized in the value descriptor or tuple buffer — zero copy.
- **Copy** (`fetch_copy_dbval`): materializes the value into a caller-provided `DB_VALUE` (deep copy / coercion if needed).
- **List** (`fetch_val_list`): evaluates a `regu_variable_list_node` linked list in order; used for projection and assignment.
- **Arithmetic** (`fetch_peek_arith`): recursively evaluates `ARITH_TYPE` nodes — all built-in functions, operators, `NEXT_VALUE`, `CURRENT_VALUE`, `CASE`, `DECODE`, date/time arithmetic, string functions, etc.

---

## Public Entry Points

| Signature | Role |
|-----------|------|
| `int fetch_peek_dbval(THREAD_ENTRY*, regu_variable_node*, val_descr*, OID* class_oid, OID* obj_oid, QFILE_TUPLE tpl, DB_VALUE** peek_dbval)` | Zero-copy evaluate; sets `*peek_dbval` to point at result |
| `int fetch_copy_dbval(THREAD_ENTRY*, regu_variable_node*, val_descr*, OID*, OID*, QFILE_TUPLE, DB_VALUE*)` | Deep-copy evaluate into caller's `DB_VALUE` |
| `int fetch_val_list(THREAD_ENTRY*, regu_variable_list_node*, val_descr*, OID*, OID*, QFILE_TUPLE, int peek)` | Evaluate a list of regu vars; `peek=1` for peek, `=0` for copy |
| `void fetch_init_val_list(regu_variable_list_node*)` | Reset value list (clear `DB_VALUE` fields before re-evaluation) |
| `void fetch_force_not_const_recursive(regu_variable_node&)` | Strip `REGU_VARIABLE_FETCH_ALL_CONST` flag from node tree (forces re-evaluation on next fetch) |

---

## Evaluation Dispatch in `fetch_peek_dbval`

```
switch regu_var->type:
    TYPE_ATTR_ID / TYPE_SHARED_ATTR_ID / TYPE_CLASS_ATTR_ID
        → extract from heap tuple slot (obj_oid + attr descriptor)
    TYPE_CONSTANT (host var or literal)
        → read from val_descr->dbval_ptr[index]
    TYPE_INARITH / TYPE_OUTARITH
        → fetch_peek_arith(…) [recursive arith evaluation]
    TYPE_FUNC
        → qdata_evaluate_function(…) [built-in function dispatch]
    TYPE_LIST_ID
        → fetch from QFILE_LIST_ID (subquery result)
    TYPE_POSITION
        → extract column from QFILE_TUPLE by position
    TYPE_POS_VALUE
        → fetch by positional value list
    TYPE_OID_VALUE
        → return OID as DB_OID value
```

---

## `fetch_peek_arith` — Arithmetic Node Evaluation

`fetch_peek_arith` handles all `ARITH_TYPE` nodes. Key operator groups:

| Operator group | Examples |
|----------------|---------|
| String functions | `T_SUBSTRING`, `T_LPAD`, `T_RPAD`, `T_REPLACE`, `T_TRANSLATE`, `T_INSTR`, `T_MID` |
| Type conversion | `T_TO_CHAR`, `T_TO_DATE`, `T_TO_TIME`, `T_TO_TIMESTAMP`, `T_TO_NUMBER` |
| Date arithmetic | `T_DATE_ADD`, `T_DATE_SUB`, `T_ADD_MONTHS`, `T_LAST_DAY` |
| Numeric | `T_ADD`, `T_SUB`, `T_MUL`, `T_DIV`, `T_MOD`, `T_ROUND`, `T_CEIL`, `T_FLOOR`, `T_ABS` |
| Serial | `T_NEXT_VALUE`, `T_CURRENT_VALUE` → calls `xserial_get_next_value` / `xserial_get_current_value` |
| Conditional | `T_CASE`, `T_DECODE`, `T_IF`, `T_IFNULL`, `T_NVL`, `T_NVL2`, `T_COALESCE`, `T_NULLIF` |
| Window/aggregate helpers | called via sub-dispatch |
| SP invocation | `T_SP_EXEC_NO_RETURN`, `T_SP_EXEC_WITH_RETURN` → `pl_exec_sp_with_context` |

> [!key-insight]
> `T_NEXT_VALUE` (SERIAL NEXTVAL) is evaluated **inside the scan loop** via `fetch_peek_arith` → `xserial_get_next_value`. This means each qualifying row generates a serial value — if the query has a WHERE clause and some rows are filtered, serial gaps are still created. The value is memoized in `arithptr->value` for the duration of a single tuple evaluation.

---

## Recursion Depth Guard

```c
if (thread_get_recursion_depth(thread_p) > prm_get_integer_value(PRM_ID_MAX_RECURSION_SQL_DEPTH)):
    er_set(ER_ERROR_SEVERITY, …, ER_MAX_RECURSION_SQL_DEPTH, 1, max_depth)
    return error
thread_inc_recursion_depth(thread_p)
// ... evaluate ...
thread_dec_recursion_depth(thread_p)
```

`PRM_ID_MAX_RECURSION_SQL_DEPTH` (default 32) limits deeply nested arithmetic expressions to prevent stack overflow. This is relevant for user-defined functions and complex CASE chains.

---

## The `REGU_VARIABLE_FETCH_ALL_CONST` Optimization

If `REGU_VARIABLE_IS_FLAGED(regu_var, REGU_VARIABLE_FETCH_ALL_CONST)` is set, `fetch_peek_arith` immediately returns the pre-computed `arithptr->value` without re-evaluation. This is set during XASL generation for constant sub-expressions.

`fetch_force_not_const_recursive` strips this flag recursively — used before re-evaluating expressions that include non-constant inputs (e.g., when the same XASL is reused with different host variables in a session statement execute cycle).

---

## Interaction with val_descr

```c
struct val_descr {
    DB_VALUE *dbval_ptr;     // host variable array
    int dbval_cnt;           // host variable count
    DB_VALUE *sys_val_ptr;   // system values (SYSDATE, USER, etc.)
    QFILE_TUPLE_RECORD tpl_rec; // current tuple reference
};
```

`fetch_peek_dbval` reads host vars from `vd->dbval_ptr[index]` for `TYPE_CONSTANT` nodes and system values from `vd->sys_val_ptr` for `T_SYSDATE`, `T_SYSDATETIME`, `T_USER`, etc.

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Build mode | Server-side (SERVER_MODE + SA_MODE); called inside scan loops |
| Thread safety | Per-thread `thread_p` context; no global state mutation except serial |
| Memory | Peek returns pointers into existing storage — caller must NOT free peeked values; copy path allocates into caller-provided `DB_VALUE` |
| Recursion limit | `PRM_ID_MAX_RECURSION_SQL_DEPTH` (default 32) |
| Serial side-effect | `T_NEXT_VALUE` advances the serial as a side-effect; no rollback of serial values on XASL retry |

---

## Lifecycle

```
Per scan iteration (inside qexec inner loops):
    fetch_peek_dbval(thread_p, pred_expr->lhs, vd, cls_oid, obj_oid, tpl, &val)
    // evaluate predicate using fetched val
    if qualifying:
        fetch_val_list(thread_p, outptr_list->valptrp, vd, …, tpl, peek=1)
        // project columns into output tuple
```

Values fetched with peek are valid only for the duration of the current tuple (until the scan advances). Code that needs to hold a value across tuples must use `fetch_copy_dbval` or `pr_clone_value`.

---

## Related

- [[components/query-executor]] — calls `fetch_peek_dbval` and `fetch_val_list` in every scan iteration
- [[components/regu-variable]] — `REGU_VARIABLE` node structure evaluated here
- [[components/scan-manager]] — scan manager drives the tuple iteration; fetch evaluates per tuple
- [[components/xasl-predicate]] — `PRED_EXPR` tree evaluated by calling fetch on its regu variables
- [[components/query-serial]] — `xserial_get_next_value` called from `fetch_peek_arith` for `T_NEXT_VALUE`
- [[components/list-file]] — `QFILE_TUPLE` is the raw tuple pointer passed to fetch functions
- [[Memory Management Conventions]]
