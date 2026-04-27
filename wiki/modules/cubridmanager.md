---
type: stub
title: "cubridmanager (submodule)"
created: 2026-04-27
updated: 2026-04-27
tags:
  - module
  - cubrid
  - manager
  - gui
  - submodule
  - stub
status: stub
related:
  - "[[modules/_index]]"
  - "[[components/cm-common-src]]"
  - "[[sources/cubrid-manual-csql]]"
  - "[[Architecture Overview]]"
  - "[[Tech Stack]]"
  - "[[hot]]"
---

# cubridmanager (Submodule)

**Status:** Stub. Pending dedicated source ingest.

## What It Is

**CUBRID Manager** — the official Java GUI client for CUBRID administration and SQL execution. A separate Eclipse-RCP-based desktop application, not part of the database engine itself.

## Where It Lives

- **Submodule path** (when checked out): `~/dev/cubrid/cubridmanager/` (or as referenced by the main repo's `.gitmodules`)
- **Upstream**: `https://github.com/CUBRID/cubridmanager`
- **Companion server-side process**: `cub_cmserver` (the Manager-server backend that the GUI connects to). Default port `cm_port = 8001`.

## What It Does

GUI-side features:

- DB connection management (multiple hosts via "Add Host")
- SQL query editor with multi-tab + history + multi-DB result comparison
- Schema browser (tables, views, triggers, procedures, indexes)
- DBA operations: createdb / restoredb / backupdb wizards
- HA monitoring (master/slave/replica status)
- Broker monitoring (CAS counts, SQL log inspection, slow-query analysis)
- DDL audit log inspection
- TDE key-file management

Requires JRE/JDK 1.6+. Auto-update from FTP after initial install.

Compatible with CUBRID 10.0+ servers (uses the JDBC driver matching each server's version, shipped in `$CUBRID/jdbc/cubrid_jdbc.jar`).

## Server-side Counterpart

`cub_cmserver` runs on the database host as part of the standard CUBRID service. Started/stopped via `cubrid manager start/stop/status`. Default config: `$CUBRID/conf/cm.conf`. Default ACL: `cm_acl_file`.

The GUI connects to `cub_cmserver` over port 8001 (configurable via `cm_port`); the server then issues admin commands locally on the DB host.

## Why a Separate Submodule

The Manager is an Eclipse-RCP product with its own build (Maven + Tycho), release cycle, and GUI development workflow. Cross-language with the engine. Separate repo lets the GUI evolve at its own pace and ships as a downloadable binary distinct from the engine.

## CSQL Alternative

For headless / scripted DBA operations, prefer [[components/csql-shell|CSQL]] (`csql --sysadm <db>`) — it covers `;CHeckpoint`, `;Killtran`, schema introspection, system parameter set/get. See [[sources/cubrid-manual-csql]] for the full session-command reference.

## Documentation

- **End-user reference**: brief summary in [[sources/cubrid-manual-csql]] §"Management Tools"; full GUI manual hosted off-site at cubrid.org.
- **Implementation source**: pending dedicated ingest. The server-side component (`cub_cmserver`, `src/cm_common/`) is summarized in [[components/cm-common-src]].

## Related

- [[components/cm-common-src]] — server-side common libraries
- [[sources/cubrid-manual-csql]] — CSQL alternative for scripted operations
- [[components/executables]] — `cubrid manager` dispatcher
- [[modules/_index]]
- Open follow-up: dedicated source ingest (flagged in [[hot]])
