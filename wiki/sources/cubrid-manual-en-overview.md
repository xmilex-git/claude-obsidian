---
created: 2026-04-27
type: source
title: "CUBRID 11.4 English Manual — Overview"
source_path: "/home/cubrid/cubrid-manual/en/"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - reference
  - hub
related:
  - "[[sources/_index]]"
  - "[[index]]"
  - "[[Architecture Overview]]"
  - "[[Tech Stack]]"
  - "[[sources/cubrid-manual-intro]]"
  - "[[sources/cubrid-manual-install]]"
  - "[[sources/cubrid-manual-csql]]"
  - "[[sources/cubrid-manual-ha]]"
  - "[[sources/cubrid-manual-shard]]"
  - "[[sources/cubrid-manual-security]]"
  - "[[sources/cubrid-manual-admin]]"
  - "[[sources/cubrid-manual-config-params]]"
  - "[[sources/cubrid-manual-error-codes]]"
  - "[[sources/cubrid-manual-api]]"
  - "[[sources/cubrid-manual-cci]]"
  - "[[sources/cubrid-manual-jdbc]]"
  - "[[sources/cubrid-manual-pl]]"
  - "[[sources/cubrid-manual-plcsql]]"
  - "[[sources/cubrid-manual-sql-foundation]]"
  - "[[sources/cubrid-manual-sql-dml]]"
  - "[[sources/cubrid-manual-sql-ddl]]"
  - "[[sources/cubrid-manual-sql-tuning-parallel]]"
  - "[[sources/cubrid-manual-sql-functions]]"
  - "[[sources/cubrid-manual-release-notes-114]]"
---

# CUBRID 11.4 English Manual — Overview

**Ingested:** 2026-04-27
**Source location:** `/home/cubrid/cubrid-manual/en/` (Sphinx-RST documentation tree)
**Manual version:** CUBRID 11.4.0.1778 (matches engine baseline `175442fc858bd0075165729756745be6f8928036`)
**Total:** 119 RST files, ~88,270 lines, ~37 MB

## What This Is

The complete English-language **end-user manual** for CUBRID 11.4, sourced from the upstream Sphinx documentation tree. Covers everything a user, application developer, or DBA touches: install, drivers, SQL, PL/CSQL, admin utilities, HA, sharding, security, error codes, release notes.

This is the *reference companion* to the `cubrid-src-*` source-tree summaries. Where the source pages document **how the engine is implemented**, the manual pages document **what behavior is contractually guaranteed to users**. When the two diverge, that's a `[!contradiction]` to file.

> [!note]
> The manual lives outside this vault at `/home/cubrid/cubrid-manual/en/`. Wiki pages reference RST files by absolute path and section anchor. Do not copy RST content into `.raw/`.

## Top-Level Structure

| Manual chapter | Wiki source page | RST entry point | Lines |
|---|---|---|---|
| Introduction (architecture, features) | [[sources/cubrid-manual-intro]] | `intro.rst` | 284 |
| Install / upgrade / env / quickstart | [[sources/cubrid-manual-install]] | `install_upgrade.rst`, `install.rst`, `env.rst`, `upgrade.rst`, `uninstall.rst`, `quick_start.rst`, `start.rst` | 1660 |
| CSQL shell + GUI tools | [[sources/cubrid-manual-csql]] | `csql.rst`, `qrytool.rst` | 1866 |
| High availability | [[sources/cubrid-manual-ha]] | `ha.rst` | 4194 |
| Database sharding | [[sources/cubrid-manual-shard]] | `shard.rst` | 957 |
| Security (TDE/SSL/ACL/auth) | [[sources/cubrid-manual-security]] | `security.rst` | 375 |
| Admin guide (utilities, control, db_manage, scripts, troubleshoot, systemtap, ddl_audit) | [[sources/cubrid-manual-admin]] | `admin/index.rst` + 8 files | ~9100 |
| System parameter reference | [[sources/cubrid-manual-config-params]] | `admin/config.rst` | 3304 |
| Error code catalogue (792 codes) | [[sources/cubrid-manual-error-codes]] | `admin/error_log_*.rst` (7 files) | ~5627 |
| API drivers (overview + thin drivers) | [[sources/cubrid-manual-api]] | `api/index.rst`, `php`, `pdo`, `odbc`, `adodotnet`, `perl`, `python`, `ruby`, `node_js` | ~2902 |
| CCI driver (C API) | [[sources/cubrid-manual-cci]] | `api/cci.rst`, `api/cciapi.rst`, `api/cci_index.rst` | 5099 |
| JDBC driver (Java API) | [[sources/cubrid-manual-jdbc]] | `api/jdbc.rst` | 1603 |
| Stored procedures (cross-cutting + Java SP + legacy methods) | [[sources/cubrid-manual-pl]] | `pl/index.rst`, `pl_create`, `pl_call`, `pl_auth`, `pl_tcl`, `pl_tuning`, `pl_package`, `jsp`, `method` | ~3420 |
| PL/CSQL (Oracle PL/SQL-compatible language, new in 11.4) | [[sources/cubrid-manual-plcsql]] | `pl/plcsql.rst`, `plcsql_overview`, `plcsql_decl`, `plcsql_stmt`, `plcsql_expr` | ~2266 |
| SQL — types, literals, i18n, transactions, auth, catalog | [[sources/cubrid-manual-sql-foundation]] | `sql/index`, `syntax`, `identifier`, `keyword`, `comment`, `literal`, `datatype*`, `i18n*`, `transaction*`, `authorization`, `db_admin`, `user_schema`, `catalog` | ~12.6K |
| SQL — DML (query/, dblink, oodb) | [[sources/cubrid-manual-sql-dml]] | `sql/query/*.rst` (16), `sql/dblink.rst`, `sql/oodb.rst` | ~7.6K |
| SQL — DDL (schema/, trigger, partition) | [[sources/cubrid-manual-sql-ddl]] | `sql/schema/*.rst` (8), `sql/trigger.rst`, `sql/partition*` | ~5.9K |
| SQL — tuning + parallel | [[sources/cubrid-manual-sql-tuning-parallel]] | `sql/tuning*`, `sql/parallel*`, `sql/join_method.inc` | ~5.7K |
| SQL — operators + functions library | [[sources/cubrid-manual-sql-functions]] | `sql/function/*.rst` (23) | ~13.4K |
| 11.4 release notes | [[sources/cubrid-manual-release-notes-114]] | `release_note/release_note_latest_ver.rst` | 1235 |

## How to Use This Manual From the Wiki

1. **Looking up a behavior**: jump to the relevant `cubrid-manual-*` page; it has a catalog of which RST file documents what, plus the most-cited facts inline.
2. **Reading the source**: read RST files directly via absolute path (`/home/cubrid/cubrid-manual/en/<chapter>/<file>.rst`). They are immutable — never edit them.
3. **Cross-linking from a component page**: a component page may cite the manual via `> See `manual: sql/tuning.rst:2300-2400` for documented behavior` — preserve the file:line citation so re-baselining can reconcile both sides.
4. **Contradictions**: when implementation (source code) diverges from documented behavior (manual), file `> [!contradiction]` callouts on both sides.

## Top-of-Mind Facts From the Manual

- **CUBRID 11.4** = build `11.4.0.1778`. Major new features: PL/CSQL (Oracle PL/SQL-compatible), HASH JOIN, expanded result/subquery cache, parallel REDO recovery, memory monitoring (`cubrid memmon`), per-broker default ACL policy, JNI loadjava `-j`, `restore_to_newdb.sh`, auto-restart of non-HA server.
- **Engine = 3 binaries from one source** (SERVER / SA / CS). Same `~/dev/cubrid/` source compiles to `cub_server`, standalone utilities, and CS-mode clients via preprocessor guards. Manual reflects this: e.g. `cubrid createdb` and `cubrid restoredb` are SA-mode; `cubrid server start` boots `cub_server` SERVER-mode; CSQL is dual-mode.
- **3-tier topology**: application → cub_broker (port 33000 default) → CAS workers → cub_master (port 1523 default) → cub_server. JDBC/CCI/ODBC/PHP/PDO/Perl/Python/Ruby drivers are CCI-based; ADO.NET and Node.js reimplement the wire protocol in pure managed/JS.
- **PL/CSQL** is the major new language in 11.4 — Oracle PL/SQL-compatible, 110-keyword reserved-word list distinct from SQL keywords, BOOLEAN+SYS_REFCURSOR + SQL-type subset, 10 system exceptions (CASE_NOT_FOUND..ZERO_DIVIDE), Owner's Rights only.
- **HASH JOIN** is opt-in via `/*+ USE_HASH */`. Hash methods (memory/hybrid/file/skip) selected by `max_hash_list_scan_size` (default 8 MB).
- **Per-user schema** since 11.2 — table identifier max byte length dropped from 254 → 222 to make room for the schema prefix; non-DBA users see only their own schema unless the table is qualified `public.<name>`.
- **TDE**: AES (default) or ARIA, 256-bit symmetric, two-level keys (master in `<db>_keys` file up to 128 keys, data keys in volume header). `ENCRYPT=AES|ARIA` clause on `CREATE TABLE`. `_db_class.tde_algorithm` exposes per-table state.
- **HA topology**: master / slave / replica. `cubrid_ha.conf` separate from `cubrid.conf`. `ha_mode = on | off | replica`. Heartbeat via UDP `ha_port_id`. `copylogdb` + `applylogdb` per replication direction. Linux-only (no HA on Windows). SYNC vs ASYNC replication.
- **CCI = wire protocol foundation**: 95 `cci_*` C functions (`cciapi.rst:3761`); 4 connection-string syntaxes (CCI / JDBC / ODBC / ADO.NET / PDO); error code partitioning (`-20001..-20999` CCI; `-10001..-10999` CAS-via-CCI; `-21001..-21999` JDBC; below `-9999` server).
- **Default ports**: master 1523, broker query_editor 30000, broker1 33000, manager 8001, PL `stored_procedure_port` (default 0 = random; set explicitly or use UDS via `stored_procedure_uds=yes`).
- **Charset/collation**: lock at `createdb` time, can NEVER be changed afterward. UTF-8 with NFC/NFD distinct (no canonical equivalence).

## Ingest Strategy Notes

- This is a **catalog ingest** — 21 source pages documenting what the manual covers + the most-cited facts. Not a full re-ingest of 88K lines.
- The manual is the **authoritative reference for user-visible behavior**. The source-code wiki documents implementation. When they conflict, the manual generally wins for *contracts*; the source wins for *current-build behavior*. File `[!contradiction]` callouts on both sides.
- Many manual sections surface facts that fill gaps in component pages — applied as **incidental enhancements** during this ingest. See per-section pages for the list.

## Pages Created (this ingest)

22 source pages — see frontmatter `related:` block at top.

## Pages Updated

See per-section pages for incidental enhancements applied to existing component pages.

## Key Insight

The manual ⇄ source split is the natural boundary for this vault: end-user contracts on one side, implementation on the other. Anchoring both to the same baseline commit (`175442fc`) lets us reconcile drift mechanically.
