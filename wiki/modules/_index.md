---
created: 2026-04-23
type: index
title: "CUBRID Modules"
updated: 2026-04-27
tags:
  - index
  - cubrid
  - module
status: active
---

# Modules

Top-level CUBRID source modules. Each page summarizes purpose, key entry points, dependencies, and notable files.

Source: `~/dev/cubrid/`

## Core

- [[broker]] — Broker layer: connection routing between clients and DB server
- [[src]] — Core database engine source (query, storage, transaction, etc.)
- [[pl_engine]] — Stored procedure / PL engine

> [!gap] Trivial directories without dedicated module pages
> `cs/` (client/server stub), `sa/` (standalone stub), `cm_common/` (manager common libs) — these are slim header/wrapper directories, not standalone subsystems. Cross-references go to component pages instead: see [[components/cm-common-src|cm-common-src]] for the manager-common implementation, and [[Build Modes (SERVER SA CS)]] for CS / SA mode mechanics.

## Interfaces / Clients (submodules)

- [[modules/cubrid-cci|cubrid-cci]] — C Client Interface submodule
- [[modules/cubrid-jdbc|cubrid-jdbc]] — JDBC driver submodule
- [[modules/cubridmanager|cubridmanager]] — GUI manager submodule

## Build / Packaging

- [[3rdparty]] — Bundled third-party sources

> [!gap] Build / packaging directories without dedicated module pages
> `cmake/`, `debian/`, `win/` — covered inline in [[Tech Stack]] and [[components/porting|porting]]; not promoted to standalone module pages because they are mechanical configuration rather than subsystems.

## Config / Data / Misc

- [[locales]] — Locale data (collations, charsets)
- [[timezones]] — Timezone data
- [[msg]] — Message catalogs (i18n)
- [[contrib]] — Contributor tools
- [[unit_tests]] — Unit test sources

> [!gap] Trivial config/data directories
> `conf/` (default config templates), `demo/` (demodb scripts), `docs/` (in-tree docs), `include/` (public headers), `util/` (internal utilities) — content is small or auto-generated; no standalone module page is warranted. Cross-references in [[Tech Stack]] and [[Architecture Overview]] cover what little is documented.

Navigation: [[index]] | [[hot]] | [[Architecture Overview]]
