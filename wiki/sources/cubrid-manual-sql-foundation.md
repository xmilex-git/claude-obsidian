---
created: 2026-04-27
type: source
title: "CUBRID Manual — SQL Foundation (types, literals, i18n, transactions, auth, catalog)"
source_path: "/home/cubrid/cubrid-manual/en/sql/index.rst, syntax.rst, identifier.rst, keyword.rst, comment.rst, literal.rst, datatype.rst, datatype_index.rst, i18n.rst, i18n_index.rst, transaction.rst, transaction_index.rst, authorization.rst, db_admin.rst, user_schema.rst, catalog.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - sql
  - types
  - i18n
  - transactions
  - auth
  - catalog
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-sql-dml]]"
  - "[[sources/cubrid-manual-sql-ddl]]"
  - "[[sources/cubrid-manual-sql-tuning-parallel]]"
  - "[[sources/cubrid-manual-sql-functions]]"
  - "[[components/db-value]]"
  - "[[components/system-catalog]]"
  - "[[components/mvcc]]"
  - "[[components/lock-manager]]"
  - "[[components/authenticate]]"
---

# CUBRID Manual — SQL Foundation

**Ingested:** 2026-04-27
**Source files:** `sql/index.rst` (66), `syntax.rst` (18), `identifier.rst` (148), `keyword.rst` (267), `comment.rst` (22), `literal.rst` (160), `datatype.rst` (2939), `datatype_index.rst` (15), `i18n.rst` (2225), `i18n_index.rst` (27), `transaction.rst` (1762), `transaction_index.rst` (22), `authorization.rst` (448), `db_admin.rst` (15), `user_schema.rst` (86), `catalog.rst` (2297). **Total ~12.6K lines.**

## What This Covers

The **lexical and semantic foundation** of CUBRID SQL — syntax rules, identifiers, keywords, literals, the type system, internationalization (charsets/collations/timezones), transactions and locking, authorization, and the system catalog.

## Section Map

| File | Lines | Content |
|---|---|---|
| `index.rst` | 66 | Top-level toctree for SQL chapter |
| `syntax.rst` | 18 | Wrapper for identifier/keyword/comment/literal |
| `identifier.rst` | 148 | Letter-first rule, case-insensitive, max byte lengths per object class (DB=17, user=31, table/trigger/view/serial=222, column/index/constraint/SP=254 since 11.2 schema prefix); reserved-word escaping with `"`/`[]`/backticks |
| `keyword.rst` | 267 | Full ~280 reserved-word table A-Z |
| `comment.rst` | 22 | Three styles: `--`, `//`, `/* */` |
| `literal.rst` | 160 | Number/date-time/bit-string/character/collection/NULL literals; DATE/TIME/DATETIME/TIMESTAMP, TZ/LTZ; B'…', 0b…, X'…', 0x… bit-string forms |
| `datatype.rst` | 2939 | **Type reference**: numeric (SMALLINT/INT/BIGINT/NUMERIC/FLOAT/DOUBLE), date/time (DATE/TIME/TIMESTAMP/DATETIME + TZ/LTZ), BIT, CHAR/VARCHAR/NCHAR, ENUM, BLOB/CLOB, collections (SET/MULTISET/SEQUENCE), JSON, OBJECT; **explicit/implicit cast tables**; string-to-date format rules; `intl_date_lang`, `oracle_style_empty_string` semantics |
| `i18n.rst` | 2225 | Charsets (ISO-8859-1, UTF-8, EUC-KR), collations + naming (`<charset>_<lang>_<desc>...`), 25+ built-in collations, LDML compilation (`make_locale`), `cubrid synccolldb`, NFC/NFD distinct (no canonical equivalence), `SET NAMES`, charset introducers (`_utf8'...'`), coercibility rules, contraction/expansion collations, CHARSET/COLLATE modifiers, JDBC i18n, timezone library compilation from IANA |
| `transaction.rst` | 1762 | MVCC snapshot model, savepoints, **cursor holdability `HOLD_CURSORS_OVER_COMMIT`**, VACUUM workers + dropped-files tracking, **lock protocol** (granularity locking, lock modes NULL/SCH-S/IS/S/IX/BU/SIX/X/SCH-M with full compatibility & transformation matrices), **BU_LOCK for loaddb**, isolation levels (READ COMMITTED=4, REP READ=5, SERIALIZABLE=6), **unique-constraint key locking with MVCC**, deadlock detection, lock_timeout |
| `authorization.rst` | 448 | DBA/PUBLIC defaults, CREATE/ALTER/DROP USER, GROUPS/MEMBERS, GRANT/REVOKE on tables/views/procedures, WITH GRANT OPTION, schema-prefixed object access (since 11.2) |
| `db_admin.rst` | 15 | Wrapper toctree for authorization/set/kill/show |
| `user_schema.rst` | 86 | **Per-user schema (since 11.2)**: `unique_name` column in `_db_class`/`db_serial`/`db_trigger`, schema-qualified access, behaviour change vs pre-11.2 |
| `catalog.rst` | 2297 | **System catalog reference** — full schema for `_db_class`, `_db_attribute`, `_db_domain`, `_db_charset`, `_db_collation`, `_db_method*`, `_db_query_spec`, `_db_index`, `_db_index_key`, `_db_auth`, `_db_data_type`, `_db_partition`, `_db_stored_procedure*`, `_db_server`, `_db_synonym`, public views `db_user`, `db_authorization`, `db_serial`, `db_trigger`, `db_ha_apply_info`, `dual`; **data type code table (1=INTEGER … 40=JSON)**; charset code table; tde_algorithm enum (0/1/2 = NONE/AES/ARIA) |

## Key Facts

### Identifiers (since 11.2)
- **DB name** max **17 bytes**.
- **User name** max **31 bytes**.
- **Table / trigger / view / serial** max **222 bytes** — was 254 pre-11.2; the 32-byte difference makes room for the schema prefix.
- **Column / index / constraint / SP** max **254 bytes**.
- Letter-first rule. Case-insensitive.
- Reserved-word escape: `"name"`, `[name]`, `` `name` `` all valid.

### Per-user schema (since 11.2)
- `SELECT name FROM athlete LIMIT 1` from DBA fails with "Unknown class dba.athlete".
- Must qualify: `SELECT name FROM public.athlete`.
- New `unique_name` column in `_db_class`, `db_serial`, `db_trigger` carries the schema-qualified name.

### Numeric types
- **NUMERIC** default precision 15, scale 0; range 1..38; max literal precision 255.
- **FLOAT(p≤7)** = single precision; **FLOAT(p>7)** silently promoted to DOUBLE.
- **MONETARY** is **deprecated** — do not use.
- Pre-2008-R2.0: integer-overflow literals widened to NUMERIC; **since R2.0 widen to BIGINT**.
- **CHAR max chars dropped to 2048 in 11.4** (was 268,435,456). Migration: convert oversize CHAR to VARCHAR.

### Date/time types
- **DATE** allows `'0000-00-00'` literal as exception (year=0 normally invalid). Same for DATETIME.
- **TIMESTAMP** bounded to UNIX epoch (1970-01-01 to 2038-01-19 03:14:07 UTC). DATETIME is the workaround for out-of-range or millisecond precision.
- **2-digit year rule** (since 2008 R3.0): YY 00-69 → 2000-2069; YY 70-99 → 1970-1999.

### Bit / string literals
- `B'1'` is stored as `B'10000000'` (right-padded to byte boundary).
- `X'…'` similarly padded to nibble pairs.
- `_utf8'...'` charset introducer overrides default charset for the literal.

### `intl_date_lang` parameter
- Forces TO_DATE/TO_TIME/TO_DATETIME/TO_TIMESTAMP/DATE_FORMAT/TIME_FORMAT/TO_CHAR/STR_TO_DATE input strings to follow locale date format.

### `oracle_style_empty_string` (default `no`)
- When `yes`, empty string and NULL are not distinguished in string functions.

### `return_null_on_function_errors` (default `no`)
- When `yes`, all-zero date/time arguments return NULL instead of raising error.

### i18n (`i18n.rst`)
- Charset locked at `cubrid createdb` time. Cannot change later.
- **NFC and NFD are distinct** — no canonical equivalence. `'café'` (NFC) ≠ `'cafe´'` (NFD).
- 25+ built-in collations.
- LDML files in `$CUBRID/conf/cubrid_locales.txt`; build via `make_locale.sh`.
- Timezone library compiled from IANA TZ data; rebuild with `cubrid synccolldb` for new TZ release.

### Transactions (`transaction.rst`)
- **MVCC snapshot model** — readers don't block writers.
- **Cursor holdability default ON since 9.0** (`HOLD_CURSORS_OVER_COMMIT`). Pre-9.0 always closed at commit.
- **Isolation levels** (only 3 valid since 10.0): `TRAN_READ_COMMITTED=4` (default), `TRAN_REP_READ=5`, `TRAN_SERIALIZABLE=6`. Old levels (`TRAN_REP_CLASS_UNCOMMIT_INSTANCE` etc.) removed; -1157 if other values set.
- **Session timeout** (`session_state_timeout`) default 21600 s = **6 hours** — controls PREPARE / @vars / LAST_INSERT_ID / ROW_COUNT lifetime.
- **Savepoints** supported.

### Lock modes
- **9 modes**: NULL, SCH-S, IS, S, IX, BU (Bulk Update), SIX, X, SCH-M.
- **BU_LOCK** introduced 10.2 for `loaddb` — compatible only with itself + SCH_S; multiple loaddb processes can load into same table concurrently, no row locks taken.
- Full compatibility + transformation matrices in `transaction.rst:807-890`.

### VACUUM
- Lives in `src/query/`, not `src/transaction/`.
- Dropped-files tracking: FILE-ID + drop-MVCCID persisted, dropper waits for VACUUM ack before file destruction.

### Authorization
- DBA + PUBLIC users created at `createdb` time, no password.
- `CREATE USER` / `ALTER USER` / `DROP USER`.
- **Groups via membership**: `ADD MEMBERS`, `DROP MEMBERS` (NEW SQL syntax in 11.4 — was `call add_member()` method).
- `GRANT <priv> ON <obj> TO <user>` / `REVOKE`.
- `WITH GRANT OPTION` supported.
- **Schema-prefixed access since 11.2** — `public.<table>`, `<owner>.<table>`.

### System catalog (`catalog.rst`)
- Underscore-prefixed tables (`_db_*`) are **internal**; user-facing views are `db_user`, `db_authorization`, `db_serial`, `db_trigger`, `db_ha_apply_info`, `dual`.
- **Data type code table**: 1=INTEGER, 4=BIGINT, ..., **40=JSON**. The `DB_TYPE` enum is ABI-frozen on disk + XASL stream — append-only after `DB_TYPE_JSON=40`.
- **`tde_algorithm` enum**: 0=NONE, 1=AES, 2=ARIA.
- **`_db_class.unique_name`** (since 11.2) — schema-qualified name.

### NEW in 11.4 (SQL changes)
- **CHAR max chars 2048** (was 268M).
- **LOB locator path: relative not absolute**.
- **Analytic functions in UPDATE JOIN restricted** (was permitted, caused unexpected behavior).
- **`for update` on system tables/views: error**.
- **View creation: type checks deferred to runtime** (Oracle compat).
- **NULL allowed in view SELECT clause** (Oracle compat).
- **AUTO_INCREMENT + DEFAULT together: error** in CREATE/ALTER COLUMN.
- **ALTER INDEX … REBUILD with new columns: error**.
- **ROWNUM exceeding NUMERIC range: error**.
- **`db_serial.att_name` renamed → `attr_name`**.
- **`ALTER USER … ADD/DROP MEMBERS`** — new SQL syntax (was method call).
- **`ALTER SERIAL … OWNER TO`** — new SQL syntax (was `call change_serial_owner()`).
- **HASH JOIN** — opt-in via `/*+ USE_HASH */` hint.
- **LEADING hint** — finer than ORDERED for join order.
- **Extended query cache** — works in CTEs and uncorrelated subqueries.

## Cross-References

- [[components/db-value]] — DB_VALUE union representation of every type
- [[components/system-catalog]] — `_db_*` table implementation
- [[components/mvcc]] — snapshot model, dropped-files tracking
- [[components/lock-manager]] — lock modes, compatibility matrix
- [[components/deadlock-detection]] — deadlock cycle detection
- [[components/authenticate]] — user/group/GRANT implementation
- [[components/timezone-build]] · `src/timezones/` — TZ library compilation
- [[sources/cubrid-manual-sql-functions]] — type cast functions, `intl_date_lang`-aware date functions
- [[sources/cubrid-manual-sql-ddl]] — CREATE USER, GRANT/REVOKE syntax in DDL context

## Incidental Wiki Enhancements

- [[components/db-value]]: documented data type code table (1=INTEGER … 40=JSON) — DB_TYPE ABI frozen, append-only after DB_TYPE_JSON=40.
- [[components/lock-manager]]: documented BU_LOCK introduced 10.2 for loaddb; full lock compatibility + transformation matrices live in `transaction.rst:807-890`; `lock_timeout=-1` infinite, `0` no-wait.
- [[components/mvcc]]: documented isolation level enum values (4/5/6) and that pre-10.0 levels are removed; -1157 if invalid value set; VACUUM dropped-files tracking semantics.
- [[components/authenticate]]: documented schema-prefixed object access since 11.2 (`public.<table>` qualification); `ALTER USER … ADD MEMBERS` new SQL syntax in 11.4.
- [[components/system-catalog]]: documented `_db_class.unique_name` (since 11.2), `tde_algorithm` enum (0/1/2 = NONE/AES/ARIA).

## Key Insight

The SQL foundation has surprising consistency: the same lock-mode + isolation-level + MVCC model from CUBRID 10.0 has carried forward unchanged through 11.4. **The biggest 11.x changes** are: (1) per-user schema in 11.2 dropping table-name max from 254 to 222, (2) HASH JOIN + LEADING hint + expanded query cache in 11.4, (3) CHAR max characters dropped from 268M to 2048 in 11.4 (memory safety). The system catalog's underscore-prefixed `_db_*` tables are internal — user code should query `db_*` views.
