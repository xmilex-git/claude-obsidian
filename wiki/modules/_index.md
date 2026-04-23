---
type: index
title: "CUBRID Modules"
updated: 2026-04-23
tags:
  - index
  - cubrid
  - module
status: active
---

# Modules

Top-level CUBRID source modules. Each page summarizes purpose, key entry points, dependencies, and notable files.

Source: `/Users/song/DEV/cubrid/`

## Core

- [[broker]] — Broker layer: connection routing between clients and DB server
- [[cs]] — Client-Server communication layer
- [[sa]] — Standalone utility executables
- [[src]] — Core database engine source (query, storage, transaction, etc.)
- [[pl_engine]] — Stored procedure / PL engine
- [[cm_common]] — CUBRID Manager common libraries

## Interfaces / Clients

- [[cubrid-cci]] — C Client Interface submodule
- [[cubrid-jdbc]] — JDBC driver submodule
- [[cubridmanager]] — GUI manager submodule

## Build / Packaging

- [[cmake]] — CMake modules and toolchain files
- [[3rdparty]] — Bundled third-party sources
- [[debian]] — Debian packaging
- [[win]] — Windows build support

## Config / Data / Misc

- [[conf]] — Default configuration files
- [[locales]] — Locale data (collations, charsets)
- [[timezones]] — Timezone data
- [[msg]] — Message catalogs (i18n)
- [[demo]] — Demo data and scripts
- [[docs]] — In-tree documentation
- [[include]] — Public headers
- [[util]] — Internal utilities
- [[contrib]] — Contributor tools
- [[unit_tests]] — Unit test sources

Navigation: [[index]] | [[hot]] | [[Architecture Overview]]
