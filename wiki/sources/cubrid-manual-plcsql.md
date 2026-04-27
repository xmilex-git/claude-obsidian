---
created: 2026-04-27
type: source
title: "CUBRID Manual — PL/CSQL Language (pl/plcsql_*.rst)"
source_path: "/home/cubrid/cubrid-manual/en/pl/plcsql.rst, plcsql_overview.rst, plcsql_decl.rst, plcsql_stmt.rst, plcsql_expr.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - plcsql
  - oracle-compat
  - language
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-pl]]"
  - "[[sources/cubrid-src-sp]]"
  - "[[components/sp]]"
  - "[[sources/cubrid-manual-release-notes-114]]"
---

# CUBRID Manual — PL/CSQL Language Reference

**Ingested:** 2026-04-27
**Source files:** `pl/plcsql.rst` (21), `plcsql_overview.rst` (967), `plcsql_decl.rst` (361), `plcsql_stmt.rst` (623), `plcsql_expr.rst` (294). Total **2266 lines**.
**Companion page:** [[sources/cubrid-manual-pl]] for cross-cutting SP, Java SP, methods, packages, tuning, auth.

## What PL/CSQL Is

CUBRID's **Oracle PL/SQL-compatible procedural language**, **NEW in 11.4**. Lets users write stored procedures and functions in a procedural dialect alongside (or instead of) Java.

PL/CSQL = "Procedural Language extension of CUBRID SQL".

## Section Map

| File | Lines | Content |
|---|---|---|
| `plcsql.rst` | 21 | Wrapper toctree |
| `plcsql_overview.rst` | 967 | CREATE syntax, lexical rules, **PL/CSQL reserved words list (~110 keywords distinct from SQL)**, type system, %TYPE/%ROWTYPE, NUMERIC/CHAR/VARCHAR precision rules, 10 system exceptions, 4 active system params, string-comparison-uses-UTF8 rule |
| `plcsql_decl.rst` | 361 | Variable/constant/exception/cursor/inner-procedure/inner-function declarations; name-hiding rules; NOT NULL initializer requirement; cursor params (IN-only); local procedure shadowing of built-ins; mutual recursion |
| `plcsql_stmt.rst` | 623 | The 15 statement types: BLOCK, SQL, OPEN/FETCH/CLOSE, OPEN-FOR, RAISE_APPLICATION_ERROR, EXECUTE IMMEDIATE, assignment, CONTINUE/EXIT (label support), NULL, RAISE, RETURN, procedure call, IF/ELSIF/ELSE, 5 LOOP forms, CASE; **unreachable-statement compile error** |
| `plcsql_expr.rst` | 294 | Operator precedence (16 levels), `SQL%ROWCOUNT`, 4 cursor attributes (`%ISOPEN`/`%FOUND`/`%NOTFOUND`/`%ROWCOUNT`), forbidden operators inside non-static SQL |

## Key Facts

### CREATE PROCEDURE/FUNCTION (PL/CSQL form)
```
CREATE OR REPLACE PROCEDURE name (params) AS LANGUAGE PLCSQL
  -- declarations
BEGIN
  -- statements
EXCEPTION
  WHEN <exc> THEN ...
END;
```
- `LANGUAGE PLCSQL` is the discriminator (Java SP uses `LANGUAGE JAVA NAME '...'`).
- SP names cannot collide with built-ins.
- **Auto-commit forced OFF inside SP** regardless of caller session.

### Reserved-word list
- PL/CSQL has its **own ~110 keyword list** distinct from CUBRID SQL keywords.
- Up to AS/IS keyword the SQL list applies; after AS/IS the PL/CSQL list applies.
- Example: `add` as a parameter name fails because it's an SQL reserved word, even though not a PL/CSQL one.
- `AUTONOMOUS_TRANSACTION` is a **reserved word for a future feature** — currently parsed but not implemented.

### Type system
- **BOOLEAN** (PL/CSQL only — not a CUBRID SQL type).
- **SYS_REFCURSOR** (PL/CSQL only — for OPEN-FOR target).
- **Subset of SQL types**: SMALLINT, INT, BIGINT, NUMERIC, FLOAT, DOUBLE, CHAR, VARCHAR, NCHAR, VARNCHAR, DATE, TIME, TIMESTAMP, DATETIME, OBJECT.
- **Unsupported**: Collections (SET/MULTISET/LIST), BIT, ENUM, BLOB, CLOB, JSON, timezone-aware timestamps.
- **`%TYPE`** — type of a column or variable (Oracle-style).
- **`%ROWTYPE`** — record type of a table row.

### NUMERIC/CHAR/VARCHAR precision rules
- **NUMERIC** in **parameter** position uniquely means "any precision 1..38, scale 0..precision".
- **NUMERIC** in **return** position expands to `(STORED_PROCEDURE_RETURN_NUMERIC_SIZE)` — default `(38, 15)`.
- **CHAR/VARCHAR** in parameter position = "any length up to 2048 chars / 1073741823 chars" respectively.
- These positional rules are unique to PL/CSQL.

### Records (%ROWTYPE)
- Records **never become NULL** after declaration. Assigning NULL sets each field NULL but `r IS NULL` is false.
- Records of "same type" (matching field count + types, names ignored) can be compared with `=`/`!=`. Other comparison operators forbidden between records.
- Assignment is **positional and ignores field names**.
- `%ROWTYPE` may **NOT** be used as a parameter or return type of top-level SPs (because records aren't a SQL type), but **may** be used in local procedures/functions.

### 10 System Exceptions (fixed SQLCODE)
| SQLCODE | Name |
|---|---|
| 0 | CASE_NOT_FOUND |
| 1 | CURSOR_ALREADY_OPEN |
| 2 | INVALID_CURSOR |
| 3 | NO_DATA_FOUND |
| 4 | PROGRAM_ERROR |
| 5 | STORAGE_ERROR |
| 6 | SQL_ERROR |
| 7 | TOO_MANY_ROWS |
| 8 | VALUE_ERROR |
| 9 | ZERO_DIVIDE |

SQLCODE space partitioning:
- 0..999 — system exceptions
- 1000 — user-declared
- >1000 — `RAISE_APPLICATION_ERROR(code, msg)` (code MUST be > 1000)

### Static SQL embedded in PL/CSQL
- `SELECT INTO` **must return exactly one row** — 0 raises `NO_DATA_FOUND`, >1 raises `TOO_MANY_ROWS`. Same rule for EXECUTE IMMEDIATE INTO.
- **DBLink supported in Static SQL only for SELECT** — DML across DBLink must use Dynamic SQL (EXECUTE IMMEDIATE).

### Forbidden inside non-static (PL/CSQL native) SQL
- **`%`** is forbidden — use `MOD`.
- **`&&`/`||`/`!`** forbidden — use `AND`/`OR`/`NOT`.
- **`||` means string concat only** in PL/CSQL, regardless of `pipes_as_concat`.
- **`+` is concat for strings** in PL/CSQL, regardless of `plus_as_concat`.
- **Backslash is literal** regardless of `no_backslash_escapes`.
- **Built-in `IF` function disallowed in PL/CSQL** because of syntactic conflict with the IF/ELSIF/ELSE statement.

### String comparison
- **In non-static PL/CSQL: UTF-8 lexicographic on Unicode codepoints regardless of DB charset/collation.**
- Only Static/Dynamic SQL respects DB collation.
- This is a notable trap for non-UTF8 DBs.

### Active system parameters
**Only 4** system params take effect inside non-static/dynamic PL/CSQL:
- `compat_numeric_division_scale`
- `oracle_compat_number_behavior`
- `oracle_style_empty_string`
- `timezone`

All others are silently ignored. (Plus `pl_transaction_control` for Java SP COMMIT semantics, and `STORED_PROCEDURE_RETURN_NUMERIC_SIZE` for default NUMERIC return.)

## Statements (15 types)

1. **BLOCK** — DECLARE/BEGIN/END nested
2. **SQL** — Static SQL embedded
3. **Cursor manipulation** — OPEN, FETCH, CLOSE, OPEN-FOR
4. **RAISE_APPLICATION_ERROR(code, msg)** — code > 1000
5. **EXECUTE IMMEDIATE** — Dynamic SQL
6. **Assignment** — `:=`
7. **CONTINUE/EXIT [label] [WHEN cond]**
8. **NULL** — no-op
9. **RAISE [exc]** — bare RAISE re-raises current exception
10. **RETURN [val]**
11. **Procedure call**
12. **IF/ELSIF/ELSE/END IF**
13. **5 LOOP forms**: basic LOOP, WHILE, FOR-iter (with REVERSE/BY step; runtime VALUE_ERROR if step ≤ 0), FOR-cursor, FOR-static-sql
14. **CASE** (statement form, with CASE_NOT_FOUND raised when no WHEN matches and no ELSE)
15. **(implicit) PROCEDURE/FUNCTION calls** dispatched through CALL semantics

### Unreachable statement = compile error
- Code after an unconditional RETURN fails compilation. Not a warning.

### Loop labels
- `<<label>>` — EXIT/CONTINUE can target outer loops by label.

## Declarations

- Variable / constant / exception / cursor / inner-procedure / inner-function.
- **NOT NULL constraint requires non-null initializer.**
- **Cursor parameters are IN-only.**
- **Local procedures/functions can shadow built-in functions** — unlike top-level SPs which cannot.
- **Mutual recursion between local procedures** is allowed (forward-references resolve in same declaration block).
- Forward-reference to a name later redeclared in same block = compile error.

## Expressions

- **`SQL%ROWCOUNT`** — 1 for single-row SELECT INTO; affected-row count for DML; 0 for COMMIT/ROLLBACK; undefined on SELECT-INTO error.
- **4 cursor attributes**: `%ISOPEN`, `%FOUND`, `%NOTFOUND`, `%ROWCOUNT`. All except `%ISOPEN` raise `INVALID_CURSOR` if cursor not open and return NULL/0 before first FETCH.
- **Operator precedence**: 16 levels (manual table).
- **SQLCODE / SQLERRM** = 0 / 'no error' outside an exception block.

## DBLink

- Static SQL can only do **SELECT** across DBLink (not INSERT/UPDATE/DELETE/MERGE/REPLACE).
- For DML across DBLink, must use **EXECUTE IMMEDIATE** dynamic SQL.

## Cross-References

- [[sources/cubrid-manual-pl]] — cross-cutting SP, Java SP, legacy methods (companion)
- [[sources/cubrid-src-sp]] — `src/sp/` JNI bridge implementation
- [[components/sp]] — SP hub
- [[sources/cubrid-manual-release-notes-114]] — PL/CSQL is the headline 11.4 feature

## Incidental Wiki Enhancements

- [[components/sp]]: documented PL/CSQL's separate ~110-keyword reserved-word list, the 4 active system params inside non-static SQL, and that PL/CSQL is forced auto-commit OFF.
- [[components/sp]]: documented the 10 system exceptions with fixed SQLCODE 0-9 and the SQLCODE partitioning (0-999 system, 1000 user, >1000 RAISE_APPLICATION_ERROR).
- [[components/sp]]: documented PL/CSQL string comparison uses UTF-8 lexicographic on Unicode codepoints regardless of DB charset/collation (only Static/Dynamic SQL respects collation).
- [[components/sp]]: documented operator restrictions (`%` forbidden, `&&`/`||`/`!` forbidden, `||`/`+` forced to concat semantics, backslash literal).

## Key Insight

PL/CSQL is **Oracle PL/SQL-compatible enough to be familiar** but has CUBRID-specific gotchas: **string comparison is UTF-8 codepoint-ordered (not collation-ordered) outside Static/Dynamic SQL**, only **4 system parameters take effect**, **records never become NULL** (assigning NULL sets fields not the record), and the **`AUTONOMOUS_TRANSACTION` keyword is reserved but unimplemented**. The headline 11.4 feature.
