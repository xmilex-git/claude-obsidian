---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/string_regex.cpp"
status: active
purpose: "REGEXP/RLIKE/REGEXP_* dispatch layer; runtime engine selection between Google RE2 and C++ std::regex; compiled-pattern caching via caller-owned cub_compiled_regex*"
key_files:
  - "string_regex.cpp (dispatch + compile logic)"
  - "string_regex.hpp (cub_compiled_regex, engine API, cub_reg_traits)"
  - "string_regex_constants.hpp (engine_type, opt_flag enums)"
  - "string_regex_re2.cpp (RE2 backend implementations)"
  - "string_regex_std.cpp (std::regex backend implementations)"
related:
  - "[[components/query-string|query-string]]"
  - "[[components/query-opfunc|query-opfunc]]"
  - "[[components/xasl-predicate|xasl-predicate]]"
  - "[[dependencies/re2|re2]]"
  - "[[components/db-value|db-value]]"
tags:
  - component
  - cubrid
  - query
  - regex
created: 2026-04-23
updated: 2026-04-23
---

# `string_regex.cpp` — REGEXP / RLIKE Dispatch Layer

Routes all regular-expression SQL operations to either Google RE2 or C++ `std::regex` depending on a server parameter, and manages compiled-pattern caching across repeated evaluations of the same predicate.

## Purpose

The `cubregex` namespace provides a backend-agnostic API (`compile`, `search`, `count`, `instr`, `replace`, `substr`) that dispatches at runtime to one of two engines. The caller in [[components/query-string|query-string]] (`db_string_rlike`, `db_string_regexp_*`) passes a `cub_compiled_regex**` pointer so that successive calls on the same predicate reuse the compiled object rather than recompiling.

## Public Entry Points

| Signature | Role |
|-----------|------|
| `cubregex::compile(cr, pattern, opt_str, collation)` | Compile pattern, select engine via `PRM_ID_REGEXP_ENGINE`, skip if already compiled for same pattern+flags+codeset |
| `cubregex::search(result, reg, src)` | RLIKE / REGEXP — returns 1 if any match, 0 if none |
| `cubregex::count(result, reg, src, position)` | REGEXP_COUNT — number of non-overlapping occurrences from `position` |
| `cubregex::instr(result, reg, src, position, occurrence, return_opt)` | REGEXP_INSTR — position of N-th occurrence; `return_opt=0` → start pos, `1` → end pos |
| `cubregex::replace(result, reg, src, repl, position, occurrence)` | REGEXP_REPLACE — `occurrence=0` replaces all |
| `cubregex::substr(result, is_matched, reg, src, position, occurrence)` | REGEXP_SUBSTR — extract N-th matching substring |

### C-level wrappers (in `string_opfunc.h`)

These are the SQL function entry points called by `fetch_func_value`:

| C function | Delegates to |
|------------|-------------|
| `db_string_rlike(src, pattern, case_sensitive, comp_regex, result)` | `cubregex::compile` + `cubregex::search` |
| `db_string_regexp_count(result, args[], num_args, comp_regex)` | `cubregex::compile` + `cubregex::count` |
| `db_string_regexp_instr(result, args[], num_args, comp_regex)` | `cubregex::compile` + `cubregex::instr` |
| `db_string_regexp_like(result, args[], num_args, comp_regex)` | `cubregex::compile` + `cubregex::search` |
| `db_string_regexp_replace(result, args[], num_args, comp_regex)` | `cubregex::compile` + `cubregex::replace` |
| `db_string_regexp_substr(result, args[], num_args, comp_regex)` | `cubregex::compile` + `cubregex::substr` |

## Execution Path

```
eval_pred_rlike7() [query_evaluator.c]
  → db_string_rlike(src, pattern, case_sens, &xasl_node->rlike_eval_term.comp_regex, &result)
      → cubregex::compile(cr, pattern_str, opt_str, collation)
          → parse_match_type(opt_str)   // 'c' = case-sensitive, 'i' = insensitive
          → prm_get_integer_value(PRM_ID_REGEXP_ENGINE)   // 0=CPPSTD, 1=RE2
          → should_compile_skip(cr, ...)  // reuse if same {pattern, type, flags, codeset}
          → compile_regex_internal(cr, utf8_pattern, type, opt_flag, collation)
              → std_compile(cr, collation)   OR   re2_compile(cr)
      → cubregex::search(result, *cr, src_str)
          → cublocale::convert_string_to_utf8(utf8_src, src, codeset)
          → std_search() / re2_search()
```

## Backend Selection: RE2 vs std::regex

> [!key-insight] Two backends exist because RE2 is faster but lacks collation support
> RE2 is the default (`PRM_ID_REGEXP_ENGINE=1`) because it has guaranteed linear time complexity and no catastrophic backtracking. However, RE2 does not support locale-specific character classes or collation-aware matching (e.g., `[[:alpha:]]` with non-ASCII characters). When collation-aware regex is needed, `LIB_CPPSTD` (`engine_type=0`) is used instead, at the cost of potentially exponential matching for pathological patterns.

The selection is a **server parameter** (`PRM_ID_REGEXP_ENGINE`), not per-column or per-query. Engine type is read at compile time (inside `cubregex::compile`) and embedded into the `compiled_regex` object.

### Backend capabilities

| Feature | `LIB_RE2` | `LIB_CPPSTD` |
|---------|-----------|--------------|
| Time complexity | O(n) guaranteed | O(n·m) or worse |
| Collation-aware `[[.x.]]` syntax | No (throws `ER_FAILED`) | Yes (via `cub_reg_traits::lookup_collatename` — but throws `error_collate` intentionally to block inconsistent results) |
| Named captures, backreferences | No | Yes |
| Input encoding | UTF-8 only | `wchar_t` (wide) |
| `OPT_ICASE` | `RE2::Options::case_sensitive=false` | `std::regex_constants::icase` |

> [!warning] std::regex [[. .]] collatename syntax is intentionally broken
> `cub_reg_traits::lookup_collatename` always throws `std::regex_error(error_collate)`. This is a deliberate guard: the POSIX `[[.x.]]` collating-element syntax gives inconsistent results across GCC/MSVC/Clang implementations. CUBRID treats any pattern using it as an error.

### Syntax options

```
OPT_SYNTAX_ECMA           = 0x1   (ECMAScript regex syntax)
OPT_SYNTAX_POSIX_EXTENDED = 0x2   (POSIX ERE)
OPT_ICASE                 = 0x4   (case-insensitive)
OPT_ERROR                 = 0xf0000000  (sentinel for parse failure)
```

Default (when no `match_type` argument is given): `OPT_ICASE` only.

## Compiled-Pattern Caching

> [!key-insight] Cache is in the XASL predicate node, not a global structure
> The `cub_compiled_regex*` pointer lives in `RLIKE_EVAL_TERM::comp_regex` inside the XASL predicate node (see [[components/xasl-predicate|xasl-predicate]]). Since an XASL node is reused across rows of the same scan, the regex is compiled once per query plan and reused per-tuple. `should_compile_skip()` tests `{type, flags, pattern, codeset}` equality — if unchanged, the existing compiled object is kept.

For a **constant** pattern (the common case: `col RLIKE 'foo.*bar'`), the pattern string never changes across rows so compilation happens exactly once per query execution.

For a **non-constant** pattern (e.g., `col RLIKE other_col`), the pattern may differ per row. `should_compile_skip` will return `false` and `compile_regex_internal` will `delete cr; cr = new compiled_regex(...)` on every changed pattern. This is correct but expensive.

## Codeset / UTF-8 Normalisation

All input strings are converted to UTF-8 before matching via `cublocale::convert_string_to_utf8`. RE2 operates natively in UTF-8. `std::regex` uses `wchar_t` internally via `cub_std_regex = std::basic_regex<wchar_t, cub_reg_traits>`. The conversion round-trips through locale helpers in `locale_helper.hpp`.

## Constraints

### NULL Handling
`db_string_rlike` and all `db_string_regexp_*` functions propagate NULL on any NULL operand. An empty source string returns 0 (no match) for `search`/`count`, and NULL for `substr`.

### Memory Ownership
`cub_compiled_regex` is heap-allocated by `compile_regex_internal` (`new compiled_regex()`). The `~compiled_regex()` destructor deletes the inner `compiled_regex_object`. Ownership belongs to the XASL node; the node is freed when the query plan is retired.

### Threading
A single XASL node is used by one thread at a time (the scan thread). No locking is required. Parallel heap scans clone the XASL node, so each worker has its own `comp_regex` pointer.

### Error Model
All functions return CUBRID error codes (`ER_*`). RE2 compilation errors are mapped to `ER_REGEX_COMPILE_ERROR`. Match-type parse errors return `ER_QPROC_INVALID_PARAMETER`.

## Lifecycle

- `compile` — called on first use per XASL node, or on pattern change.
- `search/count/instr/replace/substr` — called per-tuple.
- No server-level init/destroy; pattern caches are scoped to XASL plan lifetime.

## Related

- [[components/query-string|query-string]] — `db_string_rlike`, `db_string_regexp_*` wrappers
- [[components/xasl-predicate|xasl-predicate]] — `RLIKE_EVAL_TERM::comp_regex` cache slot
- [[components/query-evaluator|query-evaluator]] — `eval_pred_rlike7` is the predicate evaluator that calls `db_string_rlike`
- [[dependencies/re2|re2]] — bundled RE2 library
- [[components/query-opfunc|query-opfunc]] — `qdata_regexp_function` dispatches to `db_string_regexp_*`
