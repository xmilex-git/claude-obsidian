---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/string_opfunc.c"
status: active
purpose: "All string, datetime-as-string, LOB, timezone, and character-set built-in functions (~28K lines); uses collation vtable for locale-aware operations"
key_files:
  - "string_opfunc.c (~28K lines)"
  - "string_opfunc.h (full public API)"
tags:
  - component
  - cubrid
  - query
  - string
related:
  - "[[components/query|query]]"
  - "[[components/query-opfunc|query-opfunc]]"
  - "[[components/query-regex|query-regex]]"
  - "[[components/query-crypto|query-crypto]]"
  - "[[components/db-value|db-value]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `string_opfunc.c` — String & Datetime Built-in Functions

The largest single file in `src/query/` (~28K lines). Implements every string manipulation, datetime formatting, LOB conversion, and character-set function callable from SQL. All routines operate on `DB_VALUE*` and write to a caller-supplied result.

> [!key-insight] Misplaced module boundary
> The header itself notes: `/* todo(rem) this doesn't belong to query module */`. `string_opfunc.c` is used on both client and server (it is not guarded by `SERVER_MODE`). Server-mode-only functions are individually guarded: e.g., `db_guid()` is `#if !defined(CS_MODE)` and `db_get_date_weekday` / `db_get_date_item` are `#if !defined(SERVER_MODE)`.

## Purpose

`string_opfunc.c` covers five functional families:

1. **String manipulation** — CONCAT (via `||` operator), SUBSTR/SUBSTRING, LPAD, RPAD, TRIM, REPLACE, TRANSLATE, INSTR, POSITION, REPEAT, ELT, INSERT, SPACE, LENGTH, CHAR_LENGTH, LOWER, UPPER, ASCII, CHR, HEX, CONV, REVERSE, FORMAT, QUOTE, BASE64, INDEX_PREFIX, ESCAPE_STR.
2. **Pattern matching** — LIKE (via `qstr_eval_like`) and REGEXP dispatch (delegates to [[components/query-regex|query-regex]]).
3. **Datetime/timezone** — TO_DATE, TO_TIME, TO_TIMESTAMP, TO_DATETIME, TO_CHAR, DATE_FORMAT, TIME_FORMAT, STR_TO_DATE, ADD_MONTHS, LAST_DAY, MONTHS_BETWEEN, ADD_TIME, SYS_DATE/TIME/TIMESTAMP/DATETIME, UNIX_TIMESTAMP, FROM_UNIXTIME, DATE_ADD/DATE_SUB, DATE_DIFF, TIME_DIFF, TZ_OFFSET, FROM_TZ, NEW_TIME, CONV_TZ, EXTRACT.
4. **LOB** — BLOB_TO_BIT, BIT_TO_BLOB, BLOB_FROM_FILE, BLOB_LENGTH, CLOB_TO_CHAR, CHAR_TO_CLOB, CLOB_FROM_FILE, CLOB_LENGTH.
5. **Character-set / collation** — `db_string_convert_to`, `db_get_cs_coll_info`, `db_json_convert_to_utf8`, LIKE optimisation bounds.

## Public Entry Points (selected)

| Signature | Role |
|-----------|------|
| `db_string_concatenate(s1, s2, result, data_status)` | CONCAT / `||`; NULL propagation; collation unification |
| `db_string_chr(res, v1, v2)` | CHR(n USING charset) |
| `db_string_instr(src, sub, start, result)` | INSTR — byte/char-aware via collation |
| `db_string_substring(operand, src, start, len, result)` | SUBSTR / SUBSTRING — LEADING/TRAILING/BOTH variants |
| `db_string_lower / db_string_upper` | LOWER/UPPER — locale-aware via `intl_lower_string` |
| `db_string_trim(operand, charset, src, result)` | TRIM LEADING/TRAILING/BOTH |
| `db_string_pad(operand, src, len, charset, result)` | LPAD/RPAD — fills with pad_charset pattern |
| `db_string_like(src, pattern, esc, result)` | LIKE — calls `qstr_eval_like`; collation-sensitive |
| `db_string_rlike(src, pattern, case_sensitive, comp_regex, result)` | RLIKE/REGEXP — delegates to [[components/query-regex\|query-regex]] |
| `db_string_regexp_count/instr/like/replace/substr` | REGEXP_* family — all delegate via `db_string_regexp_*` to `cubregex::` |
| `db_string_replace(src, search, repl, result)` | REPLACE — collation-aware via `qstr_replace` |
| `db_string_translate(src, from, to, result)` | TRANSLATE — character mapping |
| `db_to_char(src, format, lang, result, domain)` | TO_CHAR(datetime, 'fmt') or TO_CHAR(num, 'fmt') |
| `db_to_date / db_to_time / db_to_timestamp / db_to_datetime` | Parsing functions using TIMESTAMP_FORMAT tokens |
| `db_str_to_date(date, format, lang, result, domain)` | STR_TO_DATE — MySQL-compatible |
| `db_date_add_interval_expr / db_date_sub_interval_expr` | DATE_ADD / DATE_SUB with INTERVAL |
| `db_format(number, decimals, lang, result, domain)` | FORMAT — locale decimal thousands separator |
| `db_string_md5 / db_string_sha_one / db_string_sha_two` | MD5/SHA1/SHA2 — delegates to [[components/query-crypto\|query-crypto]] |
| `db_string_aes_encrypt / db_string_aes_decrypt` | AES encryption — delegates to [[components/query-crypto\|query-crypto]] |
| `db_string_to_base64 / db_string_from_base64` | BASE64 encode/decode |
| `db_inet_aton / db_inet_ntoa` | INET_ATON / INET_NTOA |
| `db_guid(thread_p, result)` | GUID() — `!CS_MODE` only; uses `crypt_generate_random_bytes` |
| `db_string_extract_dbval(operand, dbval, result, domain)` | EXTRACT(field FROM datetime) |
| `db_get_like_optimization_bounds(...)` | B-tree index skip-scan bound for LIKE |
| `qstr_make_typed_string(...)` | Low-level: construct DB_VALUE for a typed string in-place |

## Function Registry / Dispatch

There is no runtime function-table. Each SQL function is wired to its C function at XASL generation time (`xasl_generation.c`) and evaluated via a direct call through `fetch_func_value` in `fetch.c`. `string_opfunc.c` is a library, not a dispatcher.

## CHAR vs VARCHAR vs NCHAR Handling

```
QSTR_IS_CHAR(s)       = (s == DB_TYPE_CHAR || s == DB_TYPE_VARCHAR)
QSTR_IS_BIT(s)        = (s == DB_TYPE_BIT  || s == DB_TYPE_VARBIT)
QSTR_IS_FIXED_LENGTH(s) = (s == DB_TYPE_CHAR   || s == DB_TYPE_BIT)
QSTR_IS_VARIABLE_LENGTH(s) = (s == DB_TYPE_VARCHAR || s == DB_TYPE_VARBIT)
```

> [!key-insight] NCHAR is deprecated, not truly separate
> `DB_TYPE_NCHAR` and `DB_TYPE_VARNCHAR` exist in the `DB_TYPE` enum but the parser maps them to `DB_TYPE_CHAR` / `DB_TYPE_VARCHAR` via `#define` in `csql_grammar.y`. `string_opfunc.c` does not have dedicated NCHAR code paths — national character sets are handled by the collation layer on top of ordinary `CHAR/VARCHAR`.

## Collation-Aware Entry Points

Every string comparison and search function routes through the collation virtual table:

```c
#define QSTR_COMPARE(id, s1, n1, s2, n2, ti) \
    (LANG_GET_COLLATION(id))->fastcmp(...)

#define QSTR_MATCH(id, s1, n1, s2, n2, esc, has_last_esc, match_size) \
    (LANG_GET_COLLATION(id))->strmatch(...)
```

`LANG_GET_COLLATION(id)` returns a `LANG_COLLATION*` vtable. Each collation implements `fastcmp` (memcmp-equivalent) and `strmatch` (LIKE pattern match). This means all comparison results are collation-specific at zero extra dispatch cost (pointer call).

> [!key-insight] CONCAT collation unification
> When two strings with different collations are concatenated, `db_string_concatenate` does **not** implicitly convert either string. It computes the result collation via `LANG_GET_BINARY_COLLATION` (the less-specific of the two), marks the output with that collation, and proceeds. If the two collations are incompatible (e.g., different charsets), it returns `ER_QSTR_INCOMPATIBLE_COLLATIONS`. The result is always `DB_TYPE_VARCHAR` (variable-length) even if both inputs are CHAR.

## Memory / Ownership Semantics

> [!warning] Who owns the result string buffer?
> String functions that produce a new string typically allocate via `db_private_alloc(thread_p, size)`. The resulting `DB_VALUE` has `need_clear=true` (set by `db_make_varchar` or `qstr_make_typed_string`). The **caller** must call `pr_clear_value(result)` to free it. Failure to do so is a memory leak in the per-query private heap (reclaimed at query end via `thread_p`'s private allocator, so not a permanent leak, but wasteful mid-query).

Some functions (e.g., `db_string_trim`, `db_string_pad`) return a pointer that **aliases** the source buffer when the result is unchanged (trim of an already-trimmed string). In that case `need_clear` may be `false`. Do not free such values directly.

## Execution Path

```
qexec_execute_mainblock (query_executor.c)
  → scan_next_scan()
  → fetch_val_list()
  → fetch_peek_dbval() / fetch_func_value()
      → db_string_lower(value, result)        [example]
          → db_get_string(value) → raw char ptr
          → intl_lower_string(codeset, src, result_buf, src_len)
          → qstr_make_typed_string(DB_TYPE_VARCHAR, result, prec, ...)
```

## Constraints

### NULL Propagation
All functions check `DB_IS_NULL(arg)` before any work and return `NO_ERROR` leaving `result` as NULL. String functions with multiple arguments (INSTR, SUBSTR) NULL-propagate on **any** NULL operand.

### Error Model
Return `NO_ERROR` on success, negative error code otherwise. Error codes include: `ER_QPROC_INVALID_DATATYPE` (wrong type), `ER_QSTR_INCOMPATIBLE_COLLATIONS`, `ER_QSTR_INVALID_ESCAPE_SEQUENCE`, `ER_QPROC_STRING_SIZE_TOO_BIG`.

### Threading
No global mutable state in the arithmetic paths. `db_guid()` calls `crypt_generate_random_bytes()` which is thread-safe (uses OpenSSL RAND). Datetime functions access `PRM_ID_*` parameter globals read-only.

## Lifecycle

- Per-tuple: called during scan evaluation by `fetch.c`.
- `init_builtin_calendar_names(lld)` — called once at locale load time to populate the `LANG_LOCALE_DATA` day/month name arrays.
- No per-query caching. LIKE optimisation bounds (`db_get_like_optimization_bounds`) are computed once per query plan by the optimizer.

## Related

- [[components/query|query]] — call path hub
- [[components/query-opfunc|query-opfunc]] — `qdata_strcat_dbval` is the `||` operator entry, delegates to `db_string_concatenate`
- [[components/query-regex|query-regex]] — RLIKE/REGEXP* all call into `cubregex::` from here
- [[components/query-crypto|query-crypto]] — MD5, SHA*, AES delegates
- [[components/db-value|db-value]] — DB_VALUE and string types
- [[Memory Management Conventions]]
