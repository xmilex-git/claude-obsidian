---
created: 2026-04-27
type: source
title: "CUBRID Manual — SQL DDL (schema, trigger, partition)"
source_path: "/home/cubrid/cubrid-manual/en/sql/schema/*.rst, sql/trigger.rst, sql/partition*"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - sql
  - ddl
  - schema
  - trigger
  - partition
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-sql-foundation]]"
  - "[[sources/cubrid-manual-sql-dml]]"
  - "[[sources/cubrid-manual-sql-tuning-parallel]]"
  - "[[components/btree]]"
  - "[[components/execute-schema]]"
  - "[[components/partition-pruning]]"
  - "[[components/schema-manager]]"
---

# CUBRID Manual — SQL DDL (schema, trigger, partition)

**Ingested:** 2026-04-27
**Source files:** `sql/schema/index.rst` (14), `table_stmt.rst` (2083), `index_stmt.rst` (398), `view_stmt.rst` (381), `serial_stmt.rst` (314), `server_stmt.rst` (411), `stored_routine_stmt.rst` (239), `synonym_stmt.rst` (945), plus `sql/trigger.rst` (697), `sql/partition.rst` (701), `sql/partition_index.rst` (29). **Total ~5.9K lines.**

## Section Map

| File | Lines | Content |
|---|---|---|
| `index.rst` | 14 | Toctree |
| `table_stmt.rst` | 2083 | CREATE/ALTER/DROP/RENAME TABLE; column/table constraints; FK referential actions (CASCADE/RESTRICT/NO ACTION/SET NULL); REUSE_OID/DONT_REUSE_OID; **ENCRYPT (TDE)**; CHECK constraint parsed but ignored; AUTO_INCREMENT, DEFAULT, SHARED, ON UPDATE; **deduplicate index level (0-14 since 11.3)**; CREATE TABLE LIKE / AS SELECT |
| `index_stmt.rst` | 398 | CREATE INDEX, UNIQUE INDEX, descending, filtered (WHERE), function-based, INVISIBLE, **ONLINE PARALLEL N (1-16, 16 MB batch)**; three-stage online build; DEDUPLICATE; standalone-mode ignores ONLINE |
| `view_stmt.rst` | 381 | CREATE [OR REPLACE] VIEW/VCLASS, WITH CHECK OPTION; updatable view conditions |
| `serial_stmt.rst` | 314 | CREATE SERIAL with START WITH, INCREMENT BY, MIN/MAXVALUE (range −10^36 .. 10^37), CYCLE/NOCYCLE, CACHE n; cache values lost on crash → discontinuity; rollback does not affect cached values; **ALTER SERIAL OWNER TO (NEW 11.4)** |
| `server_stmt.rst` | 411 | CREATE/ALTER/DROP SERVER for DBLink — HOST/PORT/DBNAME/USER/PASSWORD/PROPERTIES/COMMENT |
| `stored_routine_stmt.rst` | 239 | CREATE [OR REPLACE] PROCEDURE/FUNCTION; LANGUAGE PLCSQL\|JAVA; modes IN/OUT/INOUT; AUTHID DEFINER/OWNER/CALLER/CURRENT_USER; default-arg syntax |
| `synonym_stmt.rst` | 945 | CREATE/ALTER/DROP/RENAME private SYNONYM; OR REPLACE; only tables/views supported as targets; **PUBLIC synonyms NOT yet supported** |
| `trigger.rst` | 697 | CREATE/ALTER/DROP/RENAME TRIGGER; STATUS ACTIVE/INACTIVE; PRIORITY (float, default 0); event time BEFORE/AFTER/DEFERRED; type INSERT/UPDATE/DELETE + STATEMENT variants + COMMIT/ROLLBACK user triggers; actions REJECT, INVALIDATE TRANSACTION, PRINT, INSERT/UPDATE/DELETE; recursion warnings; obj/new/old correlations |
| `partition.rst` | 701 | RANGE/HASH/LIST methods, MAXVALUE, partition expression length ≤1024 bytes, **disallowed-function list**; ALTER TABLE … REORGANIZE/ADD/COALESCE/PROMOTE/DROP/ANALYZE PARTITION; pruning per method; partition-key data type list; index/PK must include partitioning key |

## Key Facts

### CREATE TABLE
- Column constraints: NOT NULL, DEFAULT, AUTO_INCREMENT, SHARED, ON UPDATE, UNIQUE, PRIMARY KEY, FOREIGN KEY.
- Table constraints: PRIMARY KEY, FOREIGN KEY, UNIQUE, CHECK (parsed but ignored — for migration compat).
- **REUSE_OID** vs **DONT_REUSE_OID** — controls OID slot reuse for OO-style references.
- **ENCRYPT=AES|ARIA** — TDE table encryption (cannot ALTER existing table to add/remove).
- **CHECK constraints are parsed but NOT enforced** — explicit gotcha.
- **AUTO_INCREMENT + DEFAULT together: error in 11.4** (was permitted before).
- **CREATE TABLE LIKE** — schema clone.
- **CREATE TABLE AS SELECT** — data clone.
- **DEDUPLICATE level (0-14, since 11.3)** — leaf-page key compression level. 0 = pre-11.2 layout, no dedup.

### Foreign Key actions
- ON DELETE / ON UPDATE clauses: CASCADE, RESTRICT, NO ACTION, SET NULL.

### CREATE INDEX
- `CREATE [UNIQUE] INDEX <name> ON <table>(<col_list>) [WHERE <pred>] [USING ASC|DESC]`.
- **Filtered indexes** (`WHERE` clause).
- **Function-based indexes** (`ON tbl(UPPER(col))`).
- **INVISIBLE** indexes (created but not used by optimizer).
- **DEDUPLICATE** clause (per-index level override).
- **`ONLINE PARALLEL N`** (N=1..16, default 16 MB batch size) — three-stage online build:
  1. SCH_M_LOCK — add empty index entry (invisible to other txns due to MVCC snapshot)
  2. IX_LOCK — populate in 16 MB batches
  3. SCH_M_LOCK — promote to visible
- **`WITH ONLINE` ignored under standalone mode** (always single-threaded).

### CREATE VIEW (VCLASS)
- `CREATE [OR REPLACE] VIEW <name> AS SELECT ...`
- WITH CHECK OPTION.
- **Updatable view conditions**: single updatable table, no aggregates/DISTINCT/UNION.
- **NEW 11.4**: type checks deferred to runtime (Oracle compat); NULL allowed in SELECT clause.

### CREATE SERIAL (sequence)
- Range: **−10^36 .. 10^37**.
- CYCLE / NOCYCLE.
- CACHE n — pre-allocate values; **cache values lost on server crash → numbering discontinuity** (silent).
- **Rollback does NOT affect cached values** — once advanced, never reused.
- **ALTER SERIAL ... OWNER TO <user>** — NEW 11.4 SQL syntax (was `call change_serial_owner()` method).
- DBA / DBA-group required to change owner.
- `_db_class.attr_name` (note: was `att_name` pre-11.4 — renamed) holds the column reference.

### CREATE SERVER (DBLink)
```
CREATE SERVER <name>
  HOST '<host>' PORT <port> DBNAME '<db>'
  USER '<user>' PASSWORD '<pw>'
  [PROPERTIES '<key=val;key=val>']
  [COMMENT '<text>']
```
- Backed by `_db_server` catalog table.
- DBLink password crypto = **time-seeded XOR obfuscation** (NOT a real cipher) per [[hot]] note.

### CREATE PROCEDURE/FUNCTION
- See [[sources/cubrid-manual-pl]] / [[sources/cubrid-manual-plcsql]] for the language semantics.
- Syntax-level: LANGUAGE PLCSQL | JAVA NAME 'spec'.

### CREATE SYNONYM
- Private synonyms only — **PUBLIC SYNONYM NOT YET SUPPORTED**.
- Targets: tables, views.
- `OR REPLACE` allowed.

### CREATE TRIGGER
- `STATUS ACTIVE | INACTIVE`.
- `PRIORITY <float>` (default 0; higher = first).
- **Event time**: BEFORE | AFTER | DEFERRED.
- **Event type**: INSERT | UPDATE | DELETE (+ STATEMENT variants) + COMMIT/ROLLBACK user triggers.
- **Actions**: REJECT, INVALIDATE TRANSACTION, PRINT, INSERT/UPDATE/DELETE.
- Correlation names: `obj` (for self), `new`/`old` (for changed row).
- Recursion warning: trigger that fires another trigger that fires the original — guard with priority/condition.

### Partitioning
- **Three methods**: RANGE, HASH, LIST.
- **MAXVALUE** sentinel for RANGE (catch-all upper bound).
- **Partition expression length capped at 1024 bytes**.
- **Supported partition-key types**: CHAR, VARCHAR, SMALLINT, INT, BIGINT, DATE, TIME, TIMESTAMP, DATETIME. **NOT supported**: DOUBLE/FLOAT/NUMERIC.
- **Index/PK must include the partitioning key**.

### Disallowed functions in partition keys (long list)
CASE, CHARSET, CHR, COALESCE, SERIAL_*, DECODE, INCR/DECR, DRAND, GREATEST, LEAST, IF, IFNULL, INSTR, NVL/NVL2, ROWNUM, INST_NUM, USER, PRIOR, WIDTH_BUCKET — anything that can be non-deterministic or has data-dependent return values is forbidden.

### ALTER PARTITION
- ADD, DROP, REORGANIZE, COALESCE, PROMOTE, ANALYZE.

### Cross-cutting NEW in 11.4
- ALTER USER ... ADD/DROP MEMBERS (was method call).
- ALTER SERIAL ... OWNER TO (was method call).
- AUTO_INCREMENT + DEFAULT together = error.
- ALTER INDEX ... REBUILD with new columns = error.
- View creation type checks deferred to runtime; NULL allowed in SELECT.
- For-update on system tables/views = error.

## Cross-References

- [[sources/cubrid-manual-sql-foundation]] — type system, identifier rules
- [[sources/cubrid-manual-sql-dml]] — DML (queries, UPDATE, etc.)
- [[sources/cubrid-manual-sql-tuning-parallel]] — index hints, partition pruning
- [[components/btree]] — index structure, online-build protocol
- [[components/execute-schema]] — DDL execution path
- [[components/schema-manager]] — schema metadata
- [[components/partition-pruning]] — pruning logic
- [[components/dblink]] — CREATE SERVER target

## Incidental Wiki Enhancements

- [[components/btree]]: documented online index build three-stage protocol (SCH_M_LOCK add → IX_LOCK populate in 16 MB batches → SCH_M_LOCK promote); MVCC-snapshot invisibility in `_db_index` until promoted; DEDUPLICATE level 0-14 (since 11.3, level 0 = pre-11.2 layout); ONLINE PARALLEL N (1-16) ignored under standalone mode.
- [[components/execute-schema]]: documented CHECK constraint is parsed-but-not-enforced (migration compat); CREATE TABLE LIKE / AS SELECT cloning forms; REUSE_OID vs DONT_REUSE_OID semantics.
- [[components/partition-pruning]]: documented partition expression 1024-byte cap; partition-key type list (CHAR, VARCHAR, SMALLINT, INT, BIGINT, DATE, TIME, TIMESTAMP, DATETIME — NOT FLOAT/DOUBLE/NUMERIC); disallowed-function list; index/PK must include partitioning key.
- [[components/query-serial]]: documented SERIAL range −10^36..10^37; CACHE values lost on crash → numbering discontinuity; rollback does NOT affect cached values; ALTER SERIAL OWNER TO (NEW 11.4); column rename `att_name` → `attr_name` in `db_serial`.
- [[components/dblink]]: documented `CREATE SERVER` DDL, `_db_server` catalog table, password crypto = time-seeded XOR obfuscation (NOT a cipher).

## Key Insight

CUBRID DDL has **three sharp edges**: (1) **CHECK constraints parse but don't enforce** — silent gotcha for migrators from PostgreSQL/Oracle; (2) **SERIAL CACHE loses values on crash** with no warning, breaking sequence monotonicity guarantees; (3) **ONLINE INDEX PARALLEL is ignored under standalone mode**, which traps DBAs running utilities. The 11.4 wave promotes several previously-method calls to first-class SQL: `ALTER USER ADD MEMBERS`, `ALTER SERIAL OWNER TO`. PUBLIC SYNONYM is still NOT supported despite the manual having a section for it.
