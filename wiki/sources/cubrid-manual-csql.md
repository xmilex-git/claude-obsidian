---
created: 2026-04-27
type: source
title: "CUBRID Manual — CSQL Interpreter + GUI Query Tools"
source_path: "/home/cubrid/cubrid-manual/en/csql.rst, qrytool.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - csql
  - shell
  - tools
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[components/csql-shell]]"
  - "[[components/executables]]"
  - "[[components/system-parameter]]"
---

# CUBRID Manual — CSQL Interpreter + GUI Query Tools

**Ingested:** 2026-04-27
**Source files:** `csql.rst` (1624 lines), `qrytool.rst` (242 lines)

## What This Covers

`csql.rst` is the reference for the **CSQL command-line interpreter** — the canonical CLI for executing SQL against CUBRID. `qrytool.rst` covers the GUI alternatives (CUBRID Manager, Migration Toolkit) and lists all supported drivers.

## Section Map (csql.rst)

| Section | Content |
|---|---|
| **Introduction** | What CSQL is and why it exists; admin-task workflows; standalone vs CS modes |
| **Executing CSQL** | Interactive vs batch mode; standalone (SA) mode bypasses `max_clients`; syntax |
| **CSQL Options** | Full command-line options table |
| **Session Commands** | The `;<cmd>` family — buffer manipulation, query exec, txn control, pager/editor/shell, schema introspection, plan/trace, history, multi-line/single-line modes, sysadmin commands |
| **Loading Files** | `;Read`, `;Write`, `;Append` |
| **Query buffer / editor** | `;Edit`, `;List`, `;Print`, `;Clear`, `;Run`, `;Xrun` |
| **Transaction control** | `;COmmit`, `;ROllback`, `;AUtocommit`, `;CHeckpoint` (sysadm only), `;Killtran` (sysadm only) |
| **Display config** | `;STring-width`, `;COLumn-width`, `;PLan`, `;Time`, `;LINe-output`, `;TRAce` |
| **History/connect/admin** | `;HISTORYList`, `;HISTORYRead`, `;CONnect`, `;.Hist`, `;.Dump_hist`, `;.X_hist` |
| **Standalone mode** | `--SA-mode` / `-S` flag — direct DB access without server process; ignores `max_clients`, allows one connection only |
| **System administrator mode** | `--sysadm` for `;CHeckpoint` and `;Killtran` |

## CSQL Session Commands (full list)

Buffer + I/O: `;REAd`, `;Write`, `;APpend`, `;PRINT`, `;SHELL`, `;CD`, `;EXit`
Editing: `;CLear`, `;EDIT [format/fmt]`, `;LISt`
Execution: `;RUn`, `;Xrun`
Transaction: `;COmmit`, `;ROllback`, `;AUtocommit [ON|OFF]`, `;REStart`, `;CHeckpoint` (sysadm), `;Killtran` (sysadm)
Tools config: `;SHELL_Cmd`, `;EDITOR_Cmd`, `;PRINT_Cmd`, `;PAger_cmd`, `;FOrmatter_cmd`
Introspection: `;DATE`, `;DATAbase`, `;SChema <class>`, `;TRIgger [*|name]`, `;Get <param>`, `;SET <param>=<v>`
Display: `;STring-width [w]`, `;COLumn-width [name]=[w]`, `;PLan [simple/detail/off]`, `;Info <cmd>`, `;TIme [ON/OFF]`, `;LINe-output [ON/OFF]`
History: `;HISTORYList`, `;HISTORYRead <n>`, `;TRAce [ON/OFF] [text/json]`
Mode: `;SIngleline [ON|OFF]`, `;CONnect <user> [dbname | dbname@host]`
Stats (DBA only): `;.Hist [ON/OFF]`, `;.Clear_hist`, `;.Dump_hist`, `;.X_hist`
Help: `;HElp`

## Key Facts

- **Session command prefix `;`** + minimum-unique-prefix matching. Capitalized chars in help are the minimum prefix (`;COm` works for `;COmmit`).
- **Case-insensitive** session commands.
- **String/comment/identifier escape**: `';exit'` inside a string literal, comment, or quoted identifier is NOT a session command — only literal text. EXCEPTION: in non-interactive mode, a semicolon as the first character of a line is always a session command (allows `;exit` even inside a buffer-fed string).
- **Batch mode**: `csql -i <script.sql>` reads SQL from file. `-c <stmts>` runs inline statements.
- **Standalone mode (`-S` or `--SA-mode`)** bypasses `cub_server`. Used for one-shot admin tasks (createdb, restoredb, checkdb). Allowed connection count = 1.
- **System admin mode** (`--sysadm`) elevates the CSQL session to perform server-side admin operations (`;CHeckpoint`, `;Killtran`). Requires DBA group membership.
- **Auto-commit default = ON** for CSQL (since CUBRID 9.x).
- **`;Get` / `;SET` system_parameter** — non-DBA can change session-scope params; DBA can change server params.
- **`;PLan simple/detail/off`** prints the optimizer plan for the next query. `;TRAce ON [text/json]` enables full SQL trace.
- **Pager**: `;PAger_cmd more|cat|less` — works only on Linux. `less` mode supports forward/backward and pattern search.
- **`CUBRID_CSQL_*` env vars** added in 11.4 (renamed from generic `EDITOR`, `SHELL`, `FORMATTER`): `CUBRID_CSQL_EDITOR`, `CUBRID_CSQL_SHELL`, `CUBRID_CSQL_FORMATTER`. Old names still accepted for compat.
- **`--sysadm` connect** is the only way to do `;CHeckpoint` and `;Killtran` from CSQL.
- **`;REStart`** — useful after HA failover. Reconnects to the post-failover server while preserving session state.
- **`;DATAbase`** also displays HA mode (active / standby / maintenance) when DB is running.

## qrytool.rst Section Map

| Section | Content |
|---|---|
| Query Tools → CSQL | Stub pointing to `csql.rst` |
| Management Tools | CUBRID Manager (Java GUI, JRE 1.6+, port 8001) and CUBRID Migration Toolkit (JDBC-based, online + offline migrations from MySQL/Oracle/CUBRID) |
| Running SQL with CUBRID Manager | Stepwise GUI workflow: install → start service → add host → authenticate → create or connect DB → run SQL |
| Migration Toolkit | What it migrates (schema + data), CSV/SQL/loaddb output formats, online via JDBC |
| **Drivers list** | JDBC, CCI, PHP, PDO, ODBC, ADO.NET, Perl, Python, Ruby, Node.js with FTP download URLs |

## Cross-References

- [[components/csql-shell]] — implementation of CSQL binary (`src/executables/csql/`)
- [[components/executables]] — `cubrid <cmd>` dispatcher
- [[components/system-parameter]] — `;Get`/`;SET` system parameters
- [[sources/cubrid-manual-api]] — full driver matrix

## Incidental Wiki Enhancements

- [[components/csql-shell]]: added `--sysadm` mode flag and the `;.Hist` / `;.Dump_hist` DBA stats commands; documented HA `;DATAbase` output (active/standby/maintenance).
- [[components/system-parameter]]: documented that the manual's `;set name=value` CSQL session command is the third channel for parameter changes (alongside file edit and `SET SYSTEM PARAMETERS` SQL).

## Key Insight

CSQL is a thicker tool than its "simple shell" reputation suggests — it has full pager/editor integration, two distinct elevation modes (`-S` standalone vs `--sysadm` server admin), and is the canonical interface for non-GUI DBA tasks (`;Killtran`, `;CHeckpoint`, history-stats). Anyone wanting to reproduce a tricky CUBRID Manager workflow should be able to do it in CSQL.
