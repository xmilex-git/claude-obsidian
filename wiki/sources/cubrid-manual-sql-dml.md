---
created: 2026-04-27
type: source
title: "CUBRID Manual — SQL DML (queries, dblink, OODB)"
source_path: "/home/cubrid/cubrid-manual/en/sql/query/*.rst, sql/dblink.rst, sql/oodb.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - sql
  - dml
  - select
  - insert
  - update
  - delete
  - merge
  - dblink
  - oodb
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-sql-foundation]]"
  - "[[sources/cubrid-manual-sql-ddl]]"
  - "[[sources/cubrid-manual-sql-tuning-parallel]]"
  - "[[components/dblink]]"
  - "[[components/execute-statement]]"
  - "[[components/cursor]]"
  - "[[components/parser]]"
---

# CUBRID Manual — SQL DML (queries, DBLink, OODB)

**Ingested:** 2026-04-27
**Source files:** `sql/query/index.rst` (19), `select.rst` (1265), `insert.rst` (317), `update.rst` (466), `delete.rst` (189), `merge.rst` (284), `cte.rst` (308), `hq.rst` (758), `show.rst` (1943), `set.rst` (151), `kill.rst` (24), `truncate.rst` (102), `prepare.rst` (131), `call.rst` (84), `do.rst` (19), `replace.rst` (72), plus `dblink.rst` (898), `oodb.rst` (388). **Total ~7.6K lines.**

## Section Map

### sql/query/ (16 files)
| File | Lines | Content |
|---|---|---|
| `select.rst` | 1265 | SELECT — qualifier (ALL/DISTINCT/UNIQUE), FROM with subquery/derived table/CLASS metaclass/DBLINK, all join syntaxes, WHERE/GROUP BY/HAVING/ORDER BY/LIMIT/USING INDEX/FOR UPDATE; WITH ROLLUP; ORDER BY NULLS FIRST/LAST |
| `insert.rst` | 317 | INSERT VALUES/SET/SELECT, ON DUPLICATE KEY UPDATE, INSERT INTO remote@server |
| `update.rst` | 466 | Single/multi-table UPDATE, ORDER BY+LIMIT semantics (since 9.0 multi-table; since 10.0 view-with-JOIN), analytic functions in SET (single-table only — **restricted in 11.4**) |
| `delete.rst` | 189 | Single/multi-table DELETE (FROM/USING forms), remote-table support since 11.3, LIMIT for single-table |
| `merge.rst` | 284 | MERGE INTO target USING source — WHEN MATCHED (UPDATE+optional DELETE WHERE) / WHEN NOT MATCHED (INSERT WHERE), USE_UPDATE_IDX/USE_INSERT_IDX hints, deterministic constraint |
| `cte.rst` | 308 | WITH … AS (CTE), RECURSIVE CTEs (single recursive allowed, must precede non-recursive in same WITH), CTE chaining; nested WITH disallowed |
| `hq.rst` | 758 | Hierarchical query — START WITH/CONNECT BY/PRIOR, NOCYCLE, pseudo-cols (LEVEL, CONNECT_BY_ISCYCLE, CONNECT_BY_ISLEAF), SYS_CONNECT_BY_PATH, ORDER SIBLINGS BY |
| `show.rst` | 1943 | DESC, EXPLAIN, SHOW TABLES/COLUMNS/INDEX/CREATE TABLE/COLLATION/TIMEZONES/GRANTS/STATISTICS/LOG STATISTICS/HEAP CAPACITY/JOB QUEUE — extensive metadata introspection |
| `set.rst` | 151 | SET SYSTEM PARAMETERS, user-defined session vars (`SET @x=…`, `:=`); 20-var cap per session |
| `kill.rst` | 24 | KILL [TRANSACTION\|QUERY] tran_index — DBA can kill any, others only own |
| `truncate.rst` | 102 | TRUNCATE [TABLE] … [CASCADE]; faster than DELETE; resets AUTO_INCREMENT; falls back to DELETE for DONT_REUSE_OID-referenced tables; disables DELETE trigger |
| `prepare.rst` | 131 | PREPARE/EXECUTE/DEALLOCATE PREPARE; max 20 prepared stmts per DB connection at SQL level |
| `call.rst` | 84 | CALL routine_name([args]) [ON [CLASS] target] [INTO :host_var] — resolution order: method (with target) → Java SP → method |
| `do.rst` | 19 | DO expression — evaluates without returning result; faster than SELECT for syntax checks |
| `replace.rst` | 72 | REPLACE INTO … VALUES/SET/SELECT — INSERT-after-DELETE on PK/UNIQUE conflict; needs INSERT+DELETE auth |

### Other DML
| File | Lines | Content |
|---|---|---|
| `dblink.rst` | 898 | CUBRID DBLink — homogeneous (CCI-based) and heterogeneous (Oracle/MySQL/MariaDB via cub_gateway/cub_cas_cgw + ODBC); `cubrid_gateway.conf`; tnsnames.ora; DBLINK SELECT and remote-table syntax in INSERT/UPDATE/DELETE/MERGE |
| `oodb.rst` | 388 | Class inheritance (super/sub class, multiple inheritance), INHERIT clause for conflict resolution, ADD/DROP SUPERCLASS, ALL/ONLY in SELECT, CLASS attributes/methods, SEQUENCE/LIST/SET path expressions; **deprecated-style OO features** |

## Key Facts

### SELECT
- Qualifiers: ALL (default), **DISTINCT**, **UNIQUE** (CUBRID-specific synonym for DISTINCT).
- **FROM** can take: table, subquery (derived table), CLASS metaclass, DBLINK clause.
- **All standard joins**: CROSS JOIN, INNER JOIN, LEFT/RIGHT OUTER JOIN, FULL OUTER JOIN.
- **ORDER BY NULLS FIRST/LAST**.
- **WITH ROLLUP** in GROUP BY.
- **FOR UPDATE** lock acquisition (X-LOCK on rows). **Restricted on system tables/views in 11.4** — error.

### UPDATE
- Multi-table UPDATE since 9.0.
- View-with-JOIN UPDATE since 10.0.
- Analytic functions in SET: single-table only. **11.4 restriction**: forbidden in UPDATE JOIN.

### DELETE
- Multi-table DELETE via FROM/USING form.
- Remote-table DELETE supported since 11.3.
- LIMIT clause: single-table only.

### MERGE
- Standard MERGE syntax.
- Hint variants `USE_UPDATE_IDX` / `USE_INSERT_IDX` for join-method control.
- **Deterministic constraint** — WHEN MATCHED clauses must determine target row uniquely.

### CTE
- `WITH cte_name AS (query) SELECT ...`
- **Recursive CTE**: `WITH RECURSIVE cte_name AS (anchor UNION ALL recursive_term)`.
- Single recursive CTE per WITH; must precede non-recursive.
- **Nested WITH disallowed**.

### Hierarchical Query
- Oracle-style `START WITH ... CONNECT BY ...`.
- Pseudo-columns: `LEVEL`, `CONNECT_BY_ISCYCLE`, `CONNECT_BY_ISLEAF`.
- Functions: `SYS_CONNECT_BY_PATH`, `PRIOR`.
- **NOCYCLE** clause to prevent infinite recursion.
- `ORDER SIBLINGS BY` for hierarchy-preserving sort.

### SHOW (extensive metadata)
- `DESC <table>` and `EXPLAIN` for plan.
- SHOW TABLES, SHOW COLUMNS, SHOW INDEX, SHOW CREATE TABLE, SHOW COLLATION, SHOW TIMEZONES, SHOW GRANTS, SHOW STATISTICS, SHOW LOG STATISTICS, SHOW HEAP CAPACITY, SHOW JOB QUEUE, etc.

### Session vars (set.rst)
- `SET @x = 5;` and `@x := 5` operator.
- **20-variable cap** per session — exceed = must DROP existing first.
- Type coercion: all stored as VARCHAR.
- **NOT rolled back on transaction abort** (per [[hot.md]]).
- Lifetime tied to `session_state_timeout` (default 21600 s = 6 hours).

### PREPARE
- `PREPARE stmt FROM 'SELECT ?'`, `EXECUTE stmt USING 5`, `DEALLOCATE PREPARE stmt`.
- **Max 20 prepared statements** per DB connection at SQL level (server memory protection).
- Broker-side `MAX_PREPARED_STMT_COUNT` (default 2000) is separate, for the client driver layer.

### TRUNCATE
- Faster than DELETE — no per-row vacuum cost.
- Resets AUTO_INCREMENT.
- **Disables DELETE trigger** (silent gotcha).
- Falls back to DELETE FROM internally if table is referenced by `DONT_REUSE_OID`.

### KILL
- `KILL [TRANSACTION|QUERY] <tran_index>`.
- DBA can kill any; others only own.

### CALL
- `CALL routine_name(args) [ON [CLASS] target] [INTO :host_var]`.
- Resolution order: **method (with target) → Java SP → method (without target)**.

### DO
- `DO expression` — evaluates without returning result row. Faster than SELECT for "just run it" cases.

### REPLACE
- `REPLACE INTO ... VALUES`, `... SET`, `... SELECT`.
- Semantics: DELETE on PK/UNIQUE conflict, then INSERT.
- Requires both INSERT and DELETE auth.

## DBLink (dblink.rst)

- **Homogeneous** = CUBRID-to-CUBRID via CCI on the CAS side.
- **Heterogeneous** = CUBRID-to-Oracle/MySQL/MariaDB via `cub_gateway` + `cub_cas_cgw` + ODBC driver.
- Configured via `CREATE SERVER` (DDL) + `cubrid_gateway.conf` + `tnsnames.ora` (Oracle case).
- DBLINK SELECT: `SELECT … FROM ... DBLINK(@server_name, 'remote_query', col_decls)`.
- Remote-table syntax: `INSERT INTO @server.table VALUES (...)`, similarly for UPDATE/DELETE/MERGE.
- 11.4: **DML across DBLink** in PL/CSQL static SQL is SELECT-only — INSERT/UPDATE/DELETE/MERGE/REPLACE require Dynamic SQL (EXECUTE IMMEDIATE).

## OODB (oodb.rst, mostly deprecated)

- Class inheritance: super/sub-classes, multiple inheritance.
- INHERIT clause for member conflict resolution.
- `ADD SUPERCLASS` / `DROP SUPERCLASS` ALTER syntax.
- `SELECT * FROM ALL <class>` vs `SELECT * FROM ONLY <class>` — full hierarchy vs single class.
- CLASS attributes (class-level vs instance-level).
- Path expressions on collection columns: `tbl.set_col[0]`, etc.
- **Largely deprecated** — most modern apps don't touch the OO surface.

## Cross-References

- [[sources/cubrid-manual-sql-foundation]] — types, literals, identifiers (companion)
- [[sources/cubrid-manual-sql-ddl]] — CREATE TABLE, CREATE INDEX, etc.
- [[sources/cubrid-manual-sql-tuning-parallel]] — query hints, optimizer behavior
- [[components/dblink]] — DBLink implementation
- [[components/execute-statement]] · [[components/execute-schema]] — DML and DDL execution paths
- [[components/cursor]] — server-side cursor for SELECT FOR UPDATE
- [[components/parser]] — SQL grammar
- [[components/parallel-query]] — parallel SELECT

## Incidental Wiki Enhancements

- [[components/dblink]]: documented two flavors — homogeneous (CCI) and heterogeneous (Oracle/MySQL/MariaDB via cub_gateway/cub_cas_cgw+ODBC); `cubrid_gateway.conf`; tnsnames.ora; DBLINK SELECT vs remote-table INSERT/UPDATE/DELETE/MERGE syntax.
- [[components/cursor]]: documented `FOR UPDATE` X-LOCK semantics; **NEW 11.4**: forbidden on system tables/views (error).
- [[components/execute-statement]]: documented MERGE deterministic constraint; TRUNCATE disables DELETE trigger silently and falls back to DELETE for DONT_REUSE_OID-referenced tables; UPDATE analytic functions restricted to single-table (forbidden in UPDATE JOIN since 11.4).
- [[components/session-variables]]: documented 20-variable per-session cap and NOT-rolled-back-on-abort semantics.

## Key Insight

CUBRID's DML is **MySQL-flavored with Oracle-isms layered on top**: ON DUPLICATE KEY UPDATE (MySQL), REPLACE INTO (MySQL), MERGE (Oracle), CONNECT BY (Oracle), CTE (standard), DBLINK with @server syntax (Oracle-ish but CUBRID-specific). The **20-prepared-statement-per-connection** limit at SQL level is a frequent gotcha for ORMs. The OODB surface is **mostly historical** — modern apps use plain relational SQL.
