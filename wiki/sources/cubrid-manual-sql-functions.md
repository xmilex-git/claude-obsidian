---
created: 2026-04-27
type: source
title: "CUBRID Manual — SQL Operators and Functions"
source_path: "/home/cubrid/cubrid-manual/en/sql/function/*.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - sql
  - functions
  - operators
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-sql-foundation]]"
  - "[[sources/cubrid-manual-sql-tuning-parallel]]"
  - "[[components/query-string]]"
  - "[[components/query-numeric]]"
  - "[[components/query-arithmetic]]"
  - "[[components/query-regex]]"
  - "[[components/query-opfunc]]"
  - "[[components/aggregate-analytic]]"
---

# CUBRID Manual — SQL Operators and Functions

**Ingested:** 2026-04-27
**Source files:** `sql/function/index.rst` (31), `logical_op.rst` (43), `comparison_op.rst` (118), `arithmetic_op.rst` (461), `set_op.rst` (169), `stmt_set_op.rst` (152), `containment_op.rst` (463), `bit_fn.rst` (120), `string_fn.rst` (2052), `regex_fn.rst` (1297), `numeric_fn.rst` (974), `datetime_fn.rst` (2366), `json_fn.rst` (1358), `typecast_fn.rst` (1538), `lob_fn.rst` (121), `analysis_fn.rst` (2130), `clickcounter_fn.rst` (108), `rownum_fn.rst` (192), `information_fn.rst` (706), `encryption_fn.rst` (116), `condition_op.rst` (464), `condition_fn.rst` (458), `other_fn.rst` (45). **Total ~13.4K lines, 23 files.**

## Function Inventory by Category

### Operators (logical_op, comparison_op, arithmetic_op, set_op, stmt_set_op, containment_op, condition_op)

**Logical** (`logical_op.rst`): AND, OR, XOR, NOT. Integer-as-boolean. **Three-valued truth table** (TRUE/FALSE/NULL). AND short-circuits on V_FALSE, NOT V_UNKNOWN (per [[hot.md]] 3VL-correct note).

**Comparison** (`comparison_op.rst`): `=`, `<=>` (NULL-safe equals), `<>`/`!=`, `>`, `<`, `>=`, `<=`, `IS [NOT] {boolean | NULL}`.

**Arithmetic** (`arithmetic_op.rst`): `+ - * / DIV % MOD`. Numeric type-cast rules. Date arithmetic (date ± integer). Set arithmetic (UNION/DIFFERENCE/INTERSECTION on collections).

**Set** (`set_op.rst`): collection arithmetic on SET/MULTISET/LIST.

**Statement set** (`stmt_set_op.rst`): UNION, DIFFERENCE/EXCEPT, INTERSECT[ION] with ALL/DISTINCT.

**Containment** (`containment_op.rst`): SETEQ, SETNEQ, SUPERSET, SUBSET, SUPERSETEQ, SUBSETEQ for collection comparison.

**Condition operators** (`condition_op.rst`): WHERE-clause expressions — basic comparison, ANY/SOME/ALL, BETWEEN, EXISTS, IN/NOT IN, LIKE, IS NULL, **CASE expression**.

**Condition functions** (`condition_fn.rst`): COALESCE, DECODE, GREATEST, IF, IFNULL, ISNULL, LEAST, NULLIF, NVL, NVL2.

### Bit (bit_fn.rst, 120 lines)
- Bitwise `&`, `|`, `^`, `~`, `<<`, `>>` — all return BIGINT.
- Aggregates: BIT_AND, BIT_OR, BIT_XOR, BIT_COUNT.

### String (string_fn.rst, 2052 lines)
- ASCII, BIN, BIT_LENGTH, CHAR_LENGTH/CHARACTER_LENGTH/LENGTH/LENGTHB
- CHR, CONCAT, CONCAT_WS, ELT, FIELD, FIND_IN_SET
- FROM_BASE64 / TO_BASE64
- INSERT, INSTR, LCASE/LOWER, LEFT, LOCATE
- LPAD / RPAD, LTRIM / RTRIM / TRIM
- MID, OCTET_LENGTH, POSITION
- REPEAT, REPLACE, REVERSE, RIGHT
- SPACE, STRCMP
- SUBSTR, SUBSTRING (FROM/FOR), SUBSTRING_INDEX
- TRANSLATE, UCASE/UPPER
- Concat operators `+`/`||` (controlled by `pipes_as_concat` / `plus_as_concat` system parameters)

### Regex (regex_fn.rst, 1297 lines)
- REGEXP, REGEXP_LIKE/RLIKE
- REGEXP_COUNT, REGEXP_INSTR
- REGEXP_REPLACE, REGEXP_SUBSTR
- **Engine**: Google RE2 (default since 11.2) or C++ `<regex>`, selectable via `regexp_engine` parameter.
- **Henry Spencer engine removed in 11.0**.
- **POSIX `[[:<:]]/[[:>:]]` no longer accepted — use `\b`**.

### Numeric (numeric_fn.rst, 974 lines)
- ABS, ACOS, ASIN, ATAN/ATAN2
- CEIL, CONV (radix conversion)
- COS/COT/SIN/TAN, DEGREES/RADIANS
- DRAND/DRANDOM, EXP, FLOOR, HEX
- LN/LOG/LOG2/LOG10, MOD, PI, POWER
- RAND/RANDOM, ROUND, SIGN, SQRT, TRUNC
- WIDTH_BUCKET

### Date/Time (datetime_fn.rst, 2366 lines — large family)
- ADDDATE/DATE_ADD, ADDTIME, ADD_MONTHS
- CURDATE/CURRENT_DATE, CURRENT_DATETIME/NOW, CURTIME/CURRENT_TIME
- CURRENT_TIMESTAMP/LOCALTIME/LOCALTIMESTAMP
- DATE, DATEDIFF, DATE_SUB/SUBDATE
- DAY/DAYOFMONTH, DAYOFWEEK, DAYOFYEAR, EXTRACT
- FROM_DAYS, FROM_TZ, FROM_UNIXTIME
- HOUR, LAST_DAY, MAKEDATE, MAKETIME
- MINUTE, MONTH, MONTHS_BETWEEN, NEW_TIME
- QUARTER, ROUND/TRUNC for date
- SEC_TO_TIME/SECOND, TIME, TIME_TO_SEC, TIMEDIFF
- TIMESTAMP, TO_DAYS, TZ_OFFSET, UNIX_TIMESTAMP
- UTC_DATE/UTC_TIME, WEEK/WEEKDAY/WEEKOFYEAR, YEAR
- `return_null_on_function_errors` semantics for zero-date inputs

### JSON (json_fn.rst, 1358 lines)
- **Operators**: JSON_ARROW (`->`), JSON_DOUBLE_ARROW (`->>`)
- **Construction**: JSON_ARRAY, JSON_ARRAYAGG (aggregate), JSON_OBJECT, JSON_OBJECTAGG (aggregate)
- **Mutation**: JSON_ARRAY_APPEND, JSON_ARRAY_INSERT, JSON_INSERT, JSON_REMOVE, JSON_REPLACE, JSON_SET
- **Inspection**: JSON_CONTAINS, JSON_CONTAINS_PATH, JSON_DEPTH, JSON_EXTRACT, JSON_KEYS, JSON_LENGTH, JSON_SEARCH, JSON_TYPE, JSON_VALID
- **Merge**: JSON_MERGE, JSON_MERGE_PATCH, JSON_MERGE_PRESERVE
- **Format**: JSON_PRETTY, JSON_QUOTE, JSON_UNQUOTE
- **Table function**: JSON_TABLE
- UTF8 codeset assumed.
- Path/pointer syntax (RFC 6901).

### Type Cast (typecast_fn.rst, 1538 lines)
- `CAST(x AS type)` with full conversion-allowed matrix.
- DATE_FORMAT, TIME_FORMAT, TO_CHAR, TO_DATE, TO_TIME, TO_DATETIME, TO_TIMESTAMP, TO_NUMBER.
- Format model: YYYY/MM/DD/HH/MI/SS/MSEC/AM/PM, etc.
- STR_TO_DATE.

### LOB (lob_fn.rst, 121 lines)
- BIT_TO_BLOB, BLOB_FROM_FILE, BLOB_LENGTH, BLOB_TO_BIT
- CHAR_TO_BLOB/CLOB, CLOB_FROM_FILE, CLOB_LENGTH, CLOB_TO_CHAR
- `local:` vs `file:` URI prefixes for server-side file access.

### Aggregate / Analytic (analysis_fn.rst, 2130 lines)
- **Aggregates**: COUNT, SUM, AVG, MIN, MAX, STDDEV*, VARIANCE*, GROUP_CONCAT, MEDIAN, JSON_ARRAYAGG, JSON_OBJECTAGG, BIT_AND/OR/XOR.
- **Analytic / window**: RANK, DENSE_RANK, PERCENT_RANK, CUME_DIST, ROW_NUMBER, NTILE, LAG, LEAD, FIRST_VALUE, LAST_VALUE, NTH_VALUE.
- `OVER (PARTITION BY ... ORDER BY ... <windowing>)` syntax.

### Click Counter (clickcounter_fn.rst, 108 lines)
- `INCR(col)`, `DECR(col)` — atomic counter, **independent of user transactions**.
- `WITH INCREMENT FOR col` shorthand in SELECT.
- Only SMALLINT / INT / BIGINT.

### Rownum (rownum_fn.rst, 192 lines)
- `ROWNUM`, `INST_NUM()`, `ORDERBY_NUM()`, `GROUPBY_NUM()`.
- Semantics differ between SELECT-list (per-row counter) and WHERE/HAVING (filter).
- `INST_NUM` == `ROWNUM` in WHERE.

### Information (information_fn.rst, 706 lines)
- CHARSET, COLLATION, COERCIBILITY
- CURRENT_USER, DATABASE/SCHEMA/DBNAME, DEFAULT
- DISK_SIZE, INDEX_CARDINALITY
- INET_*
- LAST_INSERT_ID, LIST_DBS, ROW_COUNT
- SYSTEM_USER, USER, VERSION
- Etc.

### Encryption (encryption_fn.rst, 116 lines)
- MD5, SHA1
- SHA2(str, hashlen 224|256|384|512)
- AES_ENCRYPT / AES_DECRYPT

### Other (other_fn.rst, 45 lines)
- **SLEEP(seconds)** — implemented as server-thread `usleep` (per [[hot.md]] note about `arithmetic.c`)
- SYS_GUID() — 32-char hex random

## Notable Function Behavior

### `pipes_as_concat` / `plus_as_concat`
- Default behavior: `||` is OR (logical), `+` is arithmetic.
- `pipes_as_concat=yes` makes `||` the string-concat operator (Oracle-style).
- `plus_as_concat=yes` makes `+` between strings concatenate (MySQL-style).
- **In PL/CSQL: `||` and `+` always concat for strings** regardless of these params.

### `oracle_style_empty_string`
- Default `no`. When `yes`, empty string and NULL not distinguished — affects all string functions.

### `intl_date_lang`
- Forces locale date format on all TO_DATE/TO_TIME/TO_DATETIME/TO_TIMESTAMP/DATE_FORMAT/TIME_FORMAT/TO_CHAR/STR_TO_DATE.

### `return_null_on_function_errors`
- Default `no`. When `yes`, all-zero date/time arguments return NULL instead of raising.

### `regexp_engine`
- `RE2` (default since 11.2) or `<regex>` (C++ standard).
- Spencer engine removed 11.0.

## Adding a New SQL Function (per [[hot.md]])

Registration goes through `qdata_evaluate_function` switch in `query_opfunc.c`. The `qdata_evaluate_generic_function` is a dead stub.

## Cross-References

- [[components/query-string]] — string function implementation
- [[components/query-numeric]] — numeric function implementation
- [[components/query-arithmetic]] — arithmetic operators (also owns 22 JSON scalar functions + SLEEP per hot.md)
- [[components/query-regex]] — regex engine selector
- [[components/query-opfunc]] — function dispatch table
- [[components/query-crypto]] — encryption functions
- [[components/aggregate-analytic]] — aggregate + analytic implementations
- [[components/xasl-aggregate]] · [[components/xasl-analytic]] — XASL-level support
- [[components/query-serial]] — INCR/DECR / WITH INCREMENT FOR

## Incidental Wiki Enhancements

- [[components/query-arithmetic]]: confirmed manual lists 22 JSON scalar functions + SLEEP under arithmetic_op.rst category (matches hot.md note).
- [[components/query-regex]]: documented `regexp_engine` parameter — RE2 default since 11.2, C++ `<regex>` selectable, Spencer engine removed 11.0; POSIX `[[:<:]]/[[:>:]]` no longer supported (use `\b`).
- [[components/query-opfunc]]: documented function registration via `qdata_evaluate_function` switch (per existing hot.md note); confirmed `qdata_evaluate_generic_function` is dead stub.
- [[components/aggregate-analytic]]: documented full aggregate + analytic function inventory from analysis_fn.rst.

## Key Insight

CUBRID's function library is **MySQL-compatible base + Oracle-compat extensions + JSON family + window functions**. The system parameters `oracle_style_empty_string`, `pipes_as_concat`, `plus_as_concat`, `intl_date_lang`, `return_null_on_function_errors`, and `regexp_engine` make function behavior **highly tunable per-environment** — a query that works on one CUBRID install may behave differently on another. Click Counter (`INCR`/`WITH INCREMENT FOR`) is uniquely CUBRID — atomic counters that bypass user transactions for page-view-style workloads.
