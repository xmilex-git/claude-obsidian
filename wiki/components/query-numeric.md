---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/numeric_opfunc.c"
status: active
purpose: "Fixed-point DB_NUMERIC arithmetic engine: big-integer binary arithmetic, precision/scale alignment, and bidirectional coercion with all numeric DB types"
key_files:
  - "numeric_opfunc.c"
  - "numeric_opfunc.h"
tags:
  - component
  - cubrid
  - query
  - numeric
related:
  - "[[components/query-arithmetic|query-arithmetic]]"
  - "[[components/query-opfunc|query-opfunc]]"
  - "[[components/db-value|db-value]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `numeric_opfunc.c` — DB_NUMERIC Fixed-Point Arithmetic

Implements all arithmetic and coercion operations for the `DB_NUMERIC` SQL type, which represents exact fixed-point decimal numbers up to 38 significant digits. Used whenever CUBRID must not lose precision to floating-point representation.

## Purpose

`DB_NUMERIC` is a binary big-integer stored in a 16-byte (`DB_NUMERIC_BUF_SIZE`) two's-complement little-endian byte array. The `precision` (max 38) and `scale` (decimal digits after the point) are stored in the `DB_VALUE` domain fields, not inside the buffer itself. `numeric_opfunc.c` owns all operations on this type: addition, subtraction, multiplication, division, negation, comparison, and conversion to/from int, bigint, float, double, and decimal string.

## Public Entry Points

### Arithmetic

| Signature | Role |
|-----------|------|
| `numeric_db_value_add(dbv1, dbv2, answer)` | ADD: aligns scale, calls internal big-int add, checks overflow |
| `numeric_db_value_sub(dbv1, dbv2, answer)` | SUB: align + `numeric_sub` |
| `numeric_db_value_mul(dbv1, dbv2, answer)` | MUL: doubles precision internally via `DB_LONG_NUMERIC_MULTIPLIER=2`; truncates back |
| `numeric_db_value_div(dbv1, dbv2, answer)` | DIV: long division via `numeric_long_div`; divide-by-zero → `ER_QPROC_ZERO_DIVIDE` |
| `numeric_db_value_negate(answer)` | In-place two's-complement negate |
| `numeric_db_value_abs(src, dest)` | Absolute value — strips sign bit |
| `numeric_db_value_increase(arg)` | Increment by 1 — used by FLOOR on negative NUMERIC |

### Comparison

| Signature | Role |
|-----------|------|
| `numeric_db_value_compare(dbv1, dbv2, answer)` | Aligns scales, calls `numeric_compare`; writes DB_EQ/DB_LT/DB_GT into `answer` |

### Coercion

| Signature | Role |
|-----------|------|
| `numeric_coerce_int_to_num(arg, answer)` | int → NUMERIC buffer (scale=0) |
| `numeric_coerce_bigint_to_num(arg, answer)` | DB_BIGINT → NUMERIC buffer |
| `numeric_coerce_num_to_int(arg, answer)` | NUMERIC → int (truncates fraction) |
| `numeric_coerce_num_to_bigint(arg, scale, answer)` | NUMERIC → DB_BIGINT; overflow → `ER_QPROC_OVERFLOW_COERCION` |
| `numeric_coerce_dec_str_to_num(dec_str, result)` | Decimal ASCII string → NUMERIC buffer |
| `numeric_coerce_num_to_dec_str(num, dec_str)` | NUMERIC buffer → decimal ASCII (max `NUMERIC_MAX_STRING_SIZE`=81) |
| `numeric_coerce_num_to_double(num, scale, adouble)` | NUMERIC → `double` (precision loss possible) |
| `numeric_internal_double_to_num(adouble, dst_scale, num, prec, scale)` | `double` → NUMERIC; uses `numeric_fast_convert` then full path |
| `numeric_coerce_string_to_num(astring, len, codeset, num)` | Parse decimal string into DB_VALUE NUMERIC |
| `numeric_coerce_num_to_num(src, src_prec, src_scale, dest_prec, dest_scale, dest)` | Re-scale NUMERIC to target precision/scale; may truncate |
| `numeric_db_value_coerce_to_num(src, dest, data_stat)` | Any numeric type → NUMERIC; fills `data_stat` = TRUNCATED if scale clipped |
| `numeric_db_value_coerce_from_num(src, dest, data_stat)` | NUMERIC → any numeric type |
| `numeric_db_value_coerce_from_num_strict(src, dest)` | NUMERIC → numeric; error instead of truncation |

### Inspection / Misc

| Signature | Role |
|-----------|------|
| `numeric_db_value_is_zero(arg)` | Zero predicate (byte-scan) |
| `numeric_db_value_is_positive(arg)` | Sign check |
| `numeric_db_value_print(val, buf)` | Printf-safe decimal representation |
| `numeric_init_power_value_string()` | `SERVER_MODE` init — precomputes `powers_of_2[]` and `powers_of_10[]` lookup tables |

## Execution Path

```
Query executor / fetch.c
  → qdata_add_numeric (query_opfunc.c)
      → numeric_common_prec_scale(dbv1, dbv2, &common1, &common2)
          → numeric_scale_dec() on each operand to align scales
      → numeric_add(common1.buf, common2.buf, answer.buf, DB_NUMERIC_BUF_SIZE)
      → numeric_overflow(answer.buf, prec)
          → ER_QPROC_OVERFLOW_ADDITION if exceeded
      → db_make_numeric(result, buf, prec, scale)
```

## Storage Layout

> [!key-insight] Binary big-integer, not BCD
> `DB_NUMERIC` is stored as a **16-byte two's-complement binary integer** with the LSBs in `buf[DB_NUMERIC_BUF_SIZE-1]`. It is **not** BCD. Decimal operations require binary ↔ decimal conversions via precomputed power-of-10 tables (`powers_of_10[exp][DB_NUMERIC_BUF_SIZE]`), which are populated on startup by `numeric_init_pow_of_10_helper()`. In `SERVER_MODE` this is done once in `numeric_init_power_value_string()` (mutex-guarded); in standalone mode, lazy init is used with a `bool initialized` flag.

The macro `db_locate_numeric(value)` returns `(DB_C_NUMERIC)((value)->data.num.d.buf)` — a pointer directly into the `DB_VALUE` union. No heap allocation is needed for the buffer itself.

## Precision / Scale Management

```
NUMERIC(p1, s1)  op  NUMERIC(p2, s2)
         ↓
numeric_common_prec_scale():
  common_scale = MAX(s1, s2)
  scale operands with differing scale via numeric_scale_dec()
  result_prec  = MAX(p1-s1, p2-s2) + common_scale + 1  (for carry)
  if result_prec > DB_MAX_NUMERIC_PRECISION(=38):
    numeric_prec_scale_when_overflow() shrinks scale
```

For multiplication: result scale = `s1 + s2`; if this exceeds 38 digits the scale is trimmed. The double-width intermediate uses a `DB_LONG_NUMERIC` buffer of `2 × DB_NUMERIC_BUF_SIZE` bytes allocated on the stack.

## Constraints

### NULL Handling
All `numeric_db_value_*` functions propagate NULL via `DB_IS_NULL()` checks at the top; they leave `answer` untouched and return `NO_ERROR`.

### Overflow
- Addition/subtraction: `numeric_overflow(answer, prec)` scans the top `prec`-excess bytes for non-zero; sets `ER_QPROC_OVERFLOW_ADDITION`.
- Division by zero: `numeric_is_zero(arg2)` before divide → `ER_QPROC_ZERO_DIVIDE`.
- Scale truncation on coerce: reported via `*data_stat = DATA_STATUS_TRUNCATED` (not an error; caller decides).

### Threading

- In `SERVER_MODE`, `numeric_init_power_value_string()` must be called once during server boot (protected by a POSIX mutex wrapping `pthread_once`-equivalent). All arithmetic functions are then read-only against the tables and fully re-entrant.
- In `SA_MODE`, the table is lazily initialised via `bool initialized_2 / initialized_10` (not thread-safe, but SA_MODE is single-threaded).

### Memory
All intermediate buffers (`unsigned char num[DB_NUMERIC_BUF_SIZE]`, `DEC_STRING` etc.) are stack-allocated. No heap allocation inside arithmetic paths.

## Lifecycle

- `numeric_init_power_value_string()` — called once at `SERVER_MODE` boot.
- All arithmetic functions: called per-tuple by `query_opfunc.c` or `arithmetic.c` (e.g., `db_floor_dbval` on a NUMERIC value).
- Coercion functions: called by the type system (`tp_value_auto_cast`) anywhere a type conversion is needed.

## Related

- [[components/query-arithmetic|query-arithmetic]] — calls numeric functions for FLOOR/CEIL/ROUND on NUMERIC
- [[components/query-opfunc|query-opfunc]] — `qdata_add_numeric`, `qdata_multiply_numeric`, etc. delegate here
- [[components/db-value|db-value]] — DB_VALUE container; `DB_NUMERIC_BUF_SIZE`, `DB_C_NUMERIC`
- [[Memory Management Conventions]]
