---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/query_opfunc.c"
status: active
purpose: "SERVER/SA-mode operator dispatcher: binary arithmetic (+,-,*,/), bitwise ops, string concat, tuple packing/unpacking, function evaluation switch, and CONNECT BY hierarchy traversal"
key_files:
  - "query_opfunc.c (~9.4K lines)"
  - "query_opfunc.h"
tags:
  - component
  - cubrid
  - query
  - server
related:
  - "[[components/query|query]]"
  - "[[components/query-arithmetic|query-arithmetic]]"
  - "[[components/query-numeric|query-numeric]]"
  - "[[components/query-string|query-string]]"
  - "[[components/query-regex|query-regex]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/xasl-predicate|xasl-predicate]]"
  - "[[components/db-value|db-value]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `query_opfunc.c` — Generic Operator Dispatcher

Server/SA-only file (`#error Belongs to server module` guard in header). Implements the binary arithmetic, bitwise, string concatenation, and collection operators called from `fetch.c` during REGU_VARIABLE evaluation, plus the master `qdata_evaluate_function` switch that dispatches XASL `FUNCTION_TYPE` codes to their implementations.

## Purpose

`query_opfunc.c` is the polymorphic operator layer: it knows how to add, subtract, multiply, divide, bitwise-AND/OR/XOR/SHIFT, and string-concat any pair of `DB_VALUE` types by dispatching to per-type helper functions. It also owns tuple packing/unpacking, value list management, and the `qdata_evaluate_function` switch that drives every SQL function at runtime. Conceptually it sits between `fetch.c` (REGU_VARIABLE evaluation) and the leaf implementations in [[components/query-arithmetic|query-arithmetic]], [[components/query-numeric|query-numeric]], and [[components/query-string|query-string]].

## Public Entry Points

### Binary Arithmetic (type-dispatching)

| Signature | Role |
|-----------|------|
| `qdata_add_dbval(dbval1, dbval2, res, domain)` | `+` — 30+ per-type helpers; handles numeric, time, date, sequence |
| `qdata_subtract_dbval(dbval1, dbval2, res, domain)` | `-` — full datetime subtraction (date−date → days, datetime−datetime → ms) |
| `qdata_multiply_dbval(dbval1, dbval2, res, domain)` | `*` — numeric × monetary; sequences not supported |
| `qdata_divide_dbval(dbval1, dbval2, res, domain)` | `/` — divide-by-zero check before dispatch |
| `qdata_unary_minus_dbval(res, dbval1)` | Unary `-` — overflow detection for INT_MIN, BIGINT_MIN |
| `qdata_increment_dbval(dbval1, res, incval)` | Fast increment by constant (used internally) |
| `qdata_strcat_dbval(dbval1, dbval2, res, domain)` | `||` string concat — also handles number+string and date+string via auto-cast |
| `qdata_concatenate_dbval(thread_p, dbval1, dbval2, res, domain, max_size, ctx)` | Bounded concat — checks `max_allowed_size` for aggregate string overflow |
| `qdata_extract_dbval(operand, dbval, res, domain)` | EXTRACT(field FROM datetime) — delegates to `db_string_extract_dbval` |

### Bitwise Operators

| Signature | Role |
|-----------|------|
| `qdata_bit_not_dbval(dbval, res, domain)` | `~` unary NOT — BIGINT only |
| `qdata_bit_and_dbval(dbval1, dbval2, res, domain)` | `&` — BIGINT |
| `qdata_bit_or_dbval(dbval1, dbval2, res, domain)` | `\|` — BIGINT |
| `qdata_bit_xor_dbval(dbval1, dbval2, res, domain)` | `^` — BIGINT |
| `qdata_bit_shift_dbval(dbval1, dbval2, op, res, domain)` | `<<` / `>>` (via `OPERATOR_TYPE`) — BIGINT; shift amount capped at 64 |
| `qdata_divmod_dbval(dbval1, dbval2, op, res, domain)` | DIV / MOD integer variants |

### Tuple / Value-list Management

| Signature | Role |
|-----------|------|
| `qdata_set_value_list_to_null(val_list)` | Zero out all DB_VALUEs in a VAL_LIST |
| `qdata_copy_db_value(dbval1, dbval2)` | Deep copy via `pr_clone_value` |
| `qdata_copy_db_value_to_tuple_value(dbval, tvalp, tval_size)` | Serialise DB_VALUE into tuple slot |
| `qdata_copy_valptr_list_to_tuple(thread_p, valptr_list, vd, tplrec)` | Pack full output row into QFILE_TUPLE |
| `qdata_generate_tuple_desc_for_valptr_list(...)` | Fill `QFILE_TUPLE_DESCRIPTOR` for list-file write |
| `qdata_get_single_tuple_from_list_id(thread_p, list_id, single_tuple)` | Extract one row from a singleton list |
| `qdata_get_valptr_type_list / qdata_get_val_list_type_list` | Build `qfile_tuple_value_type_list` for list-file schema |
| `qdata_tuple_to_val_list` | Deserialise QFILE_TUPLE back into VAL_LIST |

### Function Evaluation

| Signature | Role |
|-----------|------|
| `qdata_evaluate_function(thread_p, func, vd, obj_oid, tpl)` | Master switch on `FUNCTION_TYPE::ftype` (see below) |
| `qdata_regu_list_to_regu_array(...)` | Flatten operand linked list to array for varargs functions |

### Miscellaneous

| Signature | Role |
|-----------|------|
| `qdata_evaluate_connect_by_root(...)` | CONNECT_BY_ROOT — walks parent chain in list file |
| `qdata_evaluate_qprior(...)` | PRIOR in SELECT list — fetches parent tuple via list-file position |
| `qdata_evaluate_sys_connect_by_path(...)` | SYS_CONNECT_BY_PATH path accumulation |
| `qdata_list_dbs(thread_p, result, domain)` | LIST_DBS() — scans `databases.txt` |
| `qdata_get_cardinality(thread_p, class, index, key_pos, result)` | CARDINALITY(index_name) — queries btree stats |
| `qdata_apply_interpolation_function_coercion / qdata_interpolation_function_values / qdata_get_interpolation_function_result` | PERCENTILE_CONT / PERCENTILE_DISC window-function result computation |

## `qdata_evaluate_function` Switch

The master dispatcher for all `FUNCTION_TYPE` codes embedded in REGU_VARIABLE nodes of type `TYPE_FUNC`:

```
switch (funcp->ftype)
  F_SET / F_MULTISET / F_SEQUENCE / F_VID
    → qdata_convert_dbvals_to_set(...)           // build collection from operand list
  F_TABLE_SET / F_TABLE_MULTISET / F_TABLE_SEQUENCE
    → qdata_convert_table_to_set(...)            // build collection from linked subquery
  F_GENERIC
    → qdata_evaluate_generic_function(...)       // always fails (ER_QPROC_GENERIC_FUNCTION_FAILURE)
  F_CLASS_OF
    → qdata_get_class_of_function(...)           // heap_get_class_oid
  F_INSERT_SUBSTRING
    → qdata_insert_substring_function(...)
  F_ELT
    → qdata_elt(...)
  F_BENCHMARK
    → qdata_benchmark(...)
  F_JSON_ARRAY ... F_JSON_VALID (22 cases)
    → qdata_convert_operands_to_value_and_call(..., db_evaluate_json_*) // arithmetic.c
  F_REGEXP_COUNT / F_REGEXP_INSTR / F_REGEXP_LIKE / F_REGEXP_REPLACE / F_REGEXP_SUBSTR
    → qdata_regexp_function(...)                 // resolves pattern arg, calls db_string_regexp_*
  default
    → ER_QPROC_INVALID_XASLNODE
```

> [!key-insight] qdata_convert_operands_to_value_and_call is the JSON adapter
> The JSON functions have variadic signatures `(DB_VALUE* result, DB_VALUE* const* args, int num_args)`. `qdata_convert_operands_to_value_and_call` fetches all operands from `REGU_VARIABLE_LIST`, materialises them into a stack-allocated `DB_VALUE*` array, calls the target function, then bulk-clears the operand array. This avoids any heap allocation for the argument array.

## Binary Arithmetic Dispatch Pattern

Every `qdata_add_dbval`, `qdata_subtract_dbval`, `qdata_multiply_dbval` etc. follows the same dispatch pattern:

```c
switch (DB_VALUE_DOMAIN_TYPE(dbval1))
  DB_TYPE_SHORT   → qdata_add_short_to_dbval(dbval1, dbval2, result, domain)
  DB_TYPE_INTEGER → qdata_add_int_to_dbval(...)
  DB_TYPE_BIGINT  → qdata_add_bigint_to_dbval(...)
  DB_TYPE_FLOAT   → qdata_add_float_to_dbval(...)
  DB_TYPE_DOUBLE  → qdata_add_double_to_dbval(...)
  DB_TYPE_NUMERIC → qdata_add_numeric_to_dbval(...)  → numeric_db_value_add()
  DB_TYPE_MONETARY→ qdata_add_monetary_to_dbval(...)
  DB_TYPE_TIME    → qdata_add_time_to_dbval(...)
  DB_TYPE_TIMESTAMP / TIMESTAMPLTZ / TIMESTAMPTZ / DATETIME / DATETIMETZ / DATE
                  → qdata_add_*_to_dbval(...)
  DB_TYPE_SET/MULTISET/SEQUENCE → qdata_add_sequence_to_dbval(...)
  DB_TYPE_NULL / CHAR / VARCHAR / BIT / VARBIT
                  → qdata_add_chars_to_dbval(dbval1, dbval2, result)
                      → db_string_concatenate(...)   [string_opfunc.c]
```

The second operand type is handled inside each `qdata_add_<type1>_to_dbval` helper, which may further coerce the second value.

## `qdata_strcat_dbval` — String Concatenation Detail

> [!key-insight] ORACLE_STYLE_EMPTY_STRING changes NULL semantics for `||`
> Standard SQL: `NULL || 'x'` = NULL. With `PRM_ID_ORACLE_STYLE_EMPTY_STRING=true`, an empty string and NULL are treated identically for `||`, so `NULL || 'x'` = `'x'`. This mirrors Oracle behaviour and is a parameter-controlled semantic divergence.

`qdata_strcat_dbval` also handles mixed-type concat by casting numeric/datetime values to the string domain of the other operand before calling `db_string_concatenate`.

## Constraints

### NULL Propagation
The `qdata_add_dbval` family checks `DB_IS_NULL` on both operands at the top and returns `NO_ERROR` with a null result. The only exception is `qdata_strcat_dbval` when `ORACLE_STYLE_EMPTY_STRING` is active.

### Overflow Handling
- INTEGER, SHORT, BIGINT: explicit bounds checking in `qdata_add_int`, `qdata_add_bigint` etc. → `ER_QPROC_OVERFLOW_ADDITION`.
- FLOAT/DOUBLE: IEEE overflow propagated as infinity; no explicit trap.
- NUMERIC: deferred to [[components/query-numeric|query-numeric]] precision overflow check.

### Threading (SERVER_MODE guard)
The header contains `#if !defined(SERVER_MODE) && !defined(SA_MODE) #error`. All `THREAD_ENTRY*` arguments are propagated to sub-functions for private-heap allocation.

### Error Model
Returns `NO_ERROR` or a negative code. Error is set in the thread-local error context via `er_set` before returning.

## Lifecycle

- All functions are per-tuple (called from `fetch.c` per scan row).
- No query-level or session-level caching in this module.
- `qdata_benchmark` is the exception: it runs its argument expression N times in a tight loop (used for profiling SQL expressions) — can be very slow.

## Related

- [[components/query|query]] — hub
- [[components/query-arithmetic|query-arithmetic]] — JSON and scalar math functions called by `qdata_evaluate_function`
- [[components/query-numeric|query-numeric]] — `numeric_db_value_*` called for NUMERIC operands
- [[components/query-string|query-string]] — `db_string_concatenate`, `db_string_regexp_*` called from here
- [[components/regu-variable|regu-variable]] — REGU_VARIABLE / FUNCTION_TYPE; this file evaluates TYPE_FUNC nodes
- [[components/query-evaluator|query-evaluator]] — calls `qdata_add_dbval` etc. indirectly via `fetch.c`
- [[Memory Management Conventions]]
