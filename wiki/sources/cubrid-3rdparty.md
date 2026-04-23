---
type: source
title: "CUBRID 3rdparty/ — Bundled Third-Party Dependencies"
source_path: "3rdparty/"
source_type: codebase
ingested: 2026-04-23
tags:
  - source
  - cubrid
  - build
  - dependencies
related:
  - "[[modules/3rdparty|3rdparty module]]"
  - "[[dependencies/_index|Dependencies]]"
  - "[[Tech Stack]]"
---

# Source: `3rdparty/` — Bundled Third-Party Dependencies

Ingested 2026-04-23. Files read: `3rdparty/README.md`, `3rdparty/CMakeLists.txt`.

## What was read

- `README.md` — lists 8 bundled libraries with version numbers
- `CMakeLists.txt` — 556-line CMake script driving `ExternalProject_Add` for all libraries; defines download URLs, SHA-256 hashes, configure/build/install commands, and CMake variable export

## Key findings

### Library inventory (10 entries)

| Library | Pinned version | License | Mode |
|---------|---------------|---------|------|
| libexpat | 2.6.4 (tarball) / 2.2.5 (README) | MIT | EXTERNAL |
| Jansson | 2.10 | MIT | EXTERNAL |
| libedit | csql_v1.2 (CUBRID fork) | BSD-3 | EXTERNAL (Linux only) |
| OpenSSL | 1.1.1w (tarball) / 1.1.1f (README) | OpenSSL/SSLeay | EXTERNAL |
| unixODBC | 2.3.9 | LGPL-2.1 | EXTERNAL (only mode) |
| LZ4 | 1.9.4 (tarball) / 1.9.2 (README) | BSD-2 | EXTERNAL |
| RapidJSON | 1.1.0 | MIT | EXTERNAL (header-only) |
| RE2 | 2022-06-01 snapshot | BSD-3 | EXTERNAL (only mode) |
| Intel oneTBB | 2021.11.0 | Apache-2.0 | EXTERNAL (Linux only; server-only linking) |
| Flex / Bison | flex 2.6.4 / bison 3.4.1 / winflexbison 2.5.22 | LGPL/GPL+exception | SYSTEM (Linux) / EXTERNAL (Win) |

### Build architecture

- All libraries default to `EXTERNAL` mode — downloaded from `github.com/CUBRID/3rdparty` at configure time
- SHA-256 hash verification prevents unnecessary re-downloads (important for Ninja)
- Linux: all built as static libraries (`--enable-static --disable-shared`), except unixODBC (`.so`)
- Windows: prebuilt binaries from `win/3rdparty/`; DLLs copied to runtime output at build time
- Three CMake export lists: `EP_TARGETS`, `EP_LIBS`, `EP_INCLUDES` (all targets) + `TBB_TARGETS`, `TBB_LIBS`, `TBB_INCLUDES` (server-only)
- Ninja: `ADD_BY_PRODUCTS_VARIABLE` macro adds `BUILD_BYPRODUCTS` to all `ExternalProject_Add` calls

### Notable issues

- **OpenSSL 1.1.1 is EOL** since September 2023 — no upstream patches for future CVEs
- README version numbers are stale for expat (2.2.5 vs 2.6.4), OpenSSL (1.1.1f vs 1.1.1w), and LZ4 (1.9.2 vs 1.9.4); actual tarball versions are newer and authoritative
- libedit is a **CUBRID fork** (`github.com/CUBRID/libedit`) rather than upstream
- TBB allocator is disabled (`TBBMALLOC_BUILD=OFF`) to avoid conflict with CUBRID's own Heap Layers allocator
- A `FIXME` comment in CMakeLists.txt notes that the separate `TBB_*` variable set pattern should be generalized for other server-only libraries

## Pages created

- [[modules/3rdparty]] — module overview page
- [[dependencies/libexpat]] — XML parsing
- [[dependencies/jansson]] — JSON DOM manipulation
- [[dependencies/libedit]] — CLI line editing for csql
- [[dependencies/openssl]] — TLS/SSL and crypto
- [[dependencies/unixodbc]] — ODBC driver manager
- [[dependencies/lz4]] — block compression
- [[dependencies/rapidjson]] — header-only JSON parsing
- [[dependencies/re2]] — linear-time regex
- [[dependencies/libtbb]] — parallel query execution
- [[dependencies/flex-bison]] — build-time parser/lexer code generation
