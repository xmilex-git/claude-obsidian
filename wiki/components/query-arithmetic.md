---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/arithmetic.c"
status: active
purpose: "Built-in arithmetic, transcendental, JSON, and miscellaneous scalar functions; all operate on DB_VALUE*, caller owns result memory"
key_files:
  - "arithmetic.c (~6.5K lines)"
  - "arithmetic.h (full public API)"
tags:
  - component
  - cubrid
  - query
  - arithmetic
related:
  - "[[components/query|query]]"
  - "[[components/query-opfunc|query-opfunc]]"
  - "[[components/query-numeric|query-numeric]]"
  - "[[components/db-value|db-value]]"
  - "[[components/query-evaluator|query-evaluator]]"
  - "[[Memory Management Conventions]]"
  - "[[Error Handling Convention]]"
created: 2026-04-23
updated: 2026-04-23
---

# `arithmetic.c` — Arithmetic & Scalar Built-in Operators

Server-side implementations of all non-string, non-crypto scalar SQL functions. Functions take `DB_VALUE*` arguments and write into a caller-supplied `DB_VALUE* result`. No thread-local state; all functions are stateless and re-entrant.

## Purpose

`arithmetic.c` contains every pure arithmetic function callable from SQL (FLOOR, CEIL, ABS, ROUND, TRUNC, MOD, POWER, SQRT, EXP, LOG, trigonometric functions, WIDTH_BUCKET, LEAST/GREATEST, CRC32, SLEEP, and the full JSON scalar family). It does **not** contain the `+`, `-`, `*`, `/` binary operators — those live in [[components/query-opfunc|query-opfunc]] as `qdata_add_dbval`, `qdata_subtract_dbval`, etc.

## Public Entry Points

| Signature | Role |
|-----------|------|
| `db_floor_dbval(result, value)` | FLOOR — pass-through for integral types; floor() for FLOAT/DOUBLE/NUMERIC |
| `db_ceil_dbval(result, value)` | CEIL — same type dispatch as FLOOR |
| `db_abs_dbval(result, value)` | ABS — sign bit flip for INT/BIGINT/SHORT; fabs() for float types |
| `db_sign_dbval(result, value)` | SIGN → -1, 0, +1 as SHORT |
| `db_round_dbval(result, value1, value2)` | ROUND(x, d) — d=digits; dispatches to `round_date` for DATE operands |
| `db_trunc_dbval(result, value1, value2)` | TRUNC(x, d) — integer part preserving type; `truncate_date` for DATE |
| `db_mod_dbval(result, value1, value2)` | MOD — dispatches to `db_mod_<type>` helpers; strings cast to DOUBLE |
| `db_power_dbval(result, v1, v2)` | POWER — always returns DOUBLE |
| `db_sqrt_dbval(result, value)` | SQRT → DOUBLE; error on negative non-NULL |
| `db_exp_dbval(result, value)` | EXP → DOUBLE |
| `db_log_dbval(result, v1, v2)` | LOG(base, x) — arbitrary base; uses `log()` |
| `db_log_generic_dbval(result, value, b)` | LOG2 / LOG10 shim (b=2 or 10) |
| `db_sin_dbval / db_cos_dbval / db_tan_dbval` | Trig: in radians; input auto-cast to DOUBLE |
| `db_acos_dbval / db_asin_dbval / db_atan_dbval / db_atan2_dbval / db_cot_dbval` | Inverse trig / COT |
| `db_degrees_dbval / db_radians_dbval` | Conversion DEGREES ↔ RADIANS |
| `db_random_dbval(result)` | RAND() → INTEGER (calls `rand()`) |
| `db_drandom_dbval(result)` | RAND() → DOUBLE (uniform [0,1)) |
| `db_bit_count_dbval(result, value)` | BIT_COUNT — popcount on BIGINT |
| `db_typeof_dbval(result, value)` | TYPEOF — returns VARCHAR type-name string |
| `db_width_bucket(result, v1, v2, v3, v4)` | WIDTH_BUCKET(operand, min, max, num_buckets) |
| `db_sleep(result, value)` | SLEEP(seconds) — `usleep()` wrapped; returns 0 |
| `db_crc32_dbval(result, value)` | CRC32 — delegates to `crypt_crc32()` in [[components/query-crypto\|query-crypto]] |
| `db_least_or_greatest(arg1, arg2, result, least)` | LEAST / GREATEST — generic compare via `tp_value_compare` |
| `db_evaluate_json_*(result, args[], num_args)` | Full JSON scalar family (22 functions — see below) |
| `db_accumulate_json_arrayagg / db_accumulate_json_objectagg` | JSON aggregate accumulators |

### JSON function sub-family

`arithmetic.c` owns all `db_evaluate_json_*` implementations: `JSON_ARRAY`, `JSON_ARRAY_APPEND`, `JSON_ARRAY_INSERT`, `JSON_CONTAINS`, `JSON_CONTAINS_PATH`, `JSON_DEPTH`, `JSON_EXTRACT`, `JSON_GET_ALL_PATHS`, `JSON_INSERT`, `JSON_KEYS`, `JSON_LENGTH`, `JSON_MERGE_PRESERVE`, `JSON_MERGE_PATCH`, `JSON_OBJECT`, `JSON_PRETTY`, `JSON_QUOTE`, `JSON_REMOVE`, `JSON_REPLACE`, `JSON_SEARCH`, `JSON_SET`, `JSON_TYPE`, `JSON_UNQUOTE`, `JSON_VALID`.

These delegate to `src/query/db_json.hpp` and `db_json_path.hpp`.

## Execution Path

```
fetch.c: fetch_peek_dbval() / fetch_func_value()
  → arithmetic.c: db_floor_dbval(result, value)
      → switch(DB_VALUE_DOMAIN_TYPE(value))
          DB_TYPE_INTEGER  → db_make_int(result, ...)
          DB_TYPE_FLOAT    → floor(db_get_float(...))
          DB_TYPE_DOUBLE   → floor(db_get_double(...))
          DB_TYPE_NUMERIC  → numeric string manipulation
          DB_TYPE_CHAR/VARCHAR → tp_value_str_auto_cast_to_number → fallthrough to DOUBLE
          DB_TYPE_MONETARY → floor(amount)
          default          → ER_QPROC_INVALID_DATATYPE (or NULL if RETURN_NULL_ON_FUNCTION_ERRORS)
```

## Constraints

### NULL Propagation

> [!key-insight] Early-return on NULL — no result write
> Every function checks `DB_IS_NULL(value)` (or `DB_VALUE_DOMAIN_TYPE(value) == DB_TYPE_NULL`) **at the top** and returns `NO_ERROR` without writing to `result`. The result `DB_VALUE` is therefore left in whatever state the caller left it (usually initialised to `DB_TYPE_NULL` by the caller). This is the standard CUBRID three-valued-logic NULL semantics.

### Overflow Handling

- **INTEGER / SHORT overflow**: detected via range checks before the operation; returns `ER_QPROC_OVERFLOW_UMINUS`.
- **BIGINT overflow**: `DB_BIGINT_MIN` guard for unary minus (two's complement trap case).
- **FLOAT/DOUBLE overflow**: propagated as IEEE infinity; no explicit check in most functions.
- **NUMERIC overflow**: implicit in precision/scale arithmetic — if `p == DB_MAX_NUMERIC_PRECISION`, scale is decremented by 1 instead of letting p exceed the limit. See [[components/query-numeric|query-numeric]] for the detail.

### String-to-Number Auto-Cast

When a CHAR/VARCHAR argument arrives at a numeric function, `tp_value_str_auto_cast_to_number()` coerces it to DOUBLE. If that coercion fails and `PRM_ID_RETURN_NULL_ON_FUNCTION_ERRORS == true`, the function returns NULL silently. If the parameter is false, the error `ER_QPROC_INVALID_DATATYPE` is raised.

> [!warning] PRM_ID_RETURN_NULL_ON_FUNCTION_ERRORS changes semantics globally
> Setting this parameter to `true` suppresses many type errors silently as NULLs. Queries that rely on errors for data quality enforcement will stop working.

### Memory Ownership

- **Result `DB_VALUE` is caller-owned.** The functions never allocate heap memory for numeric/float results.
- **String-returning functions** (TYPEOF, JSON functions) allocate via `db_private_alloc(thread_p, ...)`. The caller must call `pr_clear_value(result)` to release.
- `arithmetic.c` does not own its `thread_p`; it propagates it for JSON allocations.

### Threading

No global mutable state. All functions are safe to call from multiple threads concurrently (each operates on caller-provided `DB_VALUE` slots on the stack or per-tuple pool).

### Error Model

Functions return `NO_ERROR` (= 0) on success or a negative error code. The result `DB_VALUE` is meaningful only on `NO_ERROR`. Callers must check before reading the result.

## Lifecycle

- Called per-tuple during scan by `fetch.c` via `fetch_func_value` / `fetch_peek_dbval`.
- No per-query or per-session initialisation.
- `db_sleep` is a side-effect function; it calls `usleep()` on the calling server thread and is therefore dangerous in high-concurrency contexts.

## Related

- [[components/query|query]] — hub; call path from scan
- [[components/query-opfunc|query-opfunc]] — binary arithmetic operators (+,-,*,/) that call different implementations
- [[components/query-numeric|query-numeric]] — DB_NUMERIC fixed-point layer used by FLOOR/CEIL/ROUND on NUMERIC type
- [[components/db-value|db-value]] — DB_VALUE container
- [[Memory Management Conventions]]
- [[Error Handling Convention]]
