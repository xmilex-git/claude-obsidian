---
created: 2026-04-23
type: index
title: "CUBRID Dependencies"
updated: 2026-04-23
tags:
  - index
  - cubrid
  - dependency
status: active
---

# Dependencies

External and bundled dependencies: version, source, license, usage inside CUBRID, and risk notes.

Source-of-truth locations: `3rdparty/`, `CMakeLists.txt`, `cmake/`, `.gitmodules`.

Navigation: [[index]] | [[modules/_index|Modules]] | [[Tech Stack]]

---

## Bundled / Downloaded (`3rdparty/`)

All libraries default to `EXTERNAL` mode — downloaded from the CUBRID-controlled GitHub mirror at configure time, SHA-256 verified. See [[modules/3rdparty|3rdparty module]] for build integration details.

| Page | Version | License | Purpose |
|------|---------|---------|---------|
| [[dependencies/libexpat\|libexpat]] | 2.6.4 | MIT | XML parsing |
| [[dependencies/jansson\|Jansson]] | 2.10 | MIT | JSON DOM manipulation |
| [[dependencies/libedit\|libedit (Editline)]] | csql_v1.2 (CUBRID fork) | BSD-3 | csql CLI line editing (Linux only) |
| [[dependencies/openssl\|OpenSSL]] | 1.1.1w | OpenSSL/SSLeay | TLS/SSL, cryptographic primitives |
| [[dependencies/unixodbc\|unixODBC]] | 2.3.9 | LGPL-2.1 | ODBC driver manager |
| [[dependencies/lz4\|LZ4]] | 1.9.4 | BSD-2 | Fast block compression |
| [[dependencies/rapidjson\|RapidJSON]] | 1.1.0 | MIT | Header-only JSON parsing |
| [[dependencies/re2\|RE2]] | 2022-06-01 | BSD-3 | Linear-time regex (REGEXP/RLIKE) |
| [[dependencies/libtbb\|Intel oneTBB]] | 2021.11.0 | Apache-2.0 | Parallel query execution (server-only) |
| [[dependencies/flex-bison\|Flex / Bison]] | flex 2.6.4 / bison 3.4.1 | LGPL/GPL+exception | Build-time parser/lexer codegen |

> [!warning] OpenSSL 1.1.1 EOL
> OpenSSL 1.1.1 reached end-of-life September 2023. No upstream security patches for new CVEs. See [[dependencies/openssl]].

---

## Vendored (in-tree, separate from `3rdparty/`)

| Page | Location | Purpose |
|------|----------|---------|
| [[components/heaplayers\|Heap Layers]] | `src/heaplayers/` | Emery Berger's composable allocator; `lea_heap.c` = dlmalloc |

---

## Submodules (`.gitmodules`)

Pending ingest.

- [[cubrid-cci]] — C Client Interface
- [[cubrid-jdbc]] — JDBC driver
- [[cubridmanager]] — GUI manager
