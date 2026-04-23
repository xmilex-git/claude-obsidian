---
type: module
path: "3rdparty/"
status: stable
purpose: "CMake ExternalProject orchestration for all third-party libraries bundled or downloaded at build time"
key_files:
  - "3rdparty/CMakeLists.txt — download URLs, SHA256 hashes, ExternalProject_Add calls, exposes EP_TARGETS / EP_LIBS / EP_INCLUDES to parent scope"
  - "3rdparty/README.md — canonical library list with versions"
tags:
  - module
  - cubrid
  - build
  - third-party
related:
  - "[[Tech Stack]]"
  - "[[Dependency Graph]]"
  - "[[dependencies/_index|Dependencies]]"
  - "[[components/heaplayers|heaplayers]]"
created: 2026-04-23
updated: 2026-04-23
---

# `3rdparty/` — Bundled & Downloaded Third-Party Libraries

> [!warning] Do not vendor-patch inside this directory
> Source tarballs are downloaded at configure time. Local edits are overwritten on a clean CMake reconfigure. Any necessary patch must be applied via `ExternalProject_Add` patch steps, tracked as an ADR.

## Purpose

`3rdparty/` is **not** a classic vendored-source directory. It contains only a `CMakeLists.txt` (and `README.md`) that drives CMake's `ExternalProject` mechanism to:

1. Download each library tarball from `https://github.com/CUBRID/3rdparty` (a CUBRID-controlled mirror), verifying SHA-256 hashes.
2. Configure, build, and install each library into `${CMAKE_BINARY_DIR}/3rdparty/` (under `lib/`, `include/`, `Source/`).
3. Export three CMake lists to the parent scope: `EP_TARGETS`, `EP_LIBS`, `EP_INCLUDES` — consumed by `cubrid/`, `cs/`, `sa/`, and `broker/` targets.
4. Export a separate `TBB_*` set (server-only linking) because TBB is only required by `libcubrid.so`.

## Build Integration

All libraries default to `EXTERNAL` mode — built from the CUBRID mirror URL. An override cache variable per library allows switching to `SYSTEM` (already-installed headers/libs) or `BUNDLED` (prebuilt, Win32 only):

| CMake variable | Default | Alternatives |
|----------------|---------|--------------|
| `WITH_LIBFLEXBISON` | `SYSTEM` (Linux) / `EXTERNAL` (Win) | — |
| `WITH_LIBEXPAT` | `EXTERNAL` | `SYSTEM` |
| `WITH_LIBJANSSON` | `EXTERNAL` | `SYSTEM` |
| `WITH_LIBEDIT` | `EXTERNAL` (Linux only) | `SYSTEM` |
| `WITH_LIBOPENSSL` | `EXTERNAL` | `SYSTEM` |
| `WITH_LIBUNIXODBC` | `EXTERNAL` | — (only EXTERNAL supported) |
| `WITH_LZ4` | `EXTERNAL` | — |
| `WITH_RE2` | `EXTERNAL` | — (only EXTERNAL supported) |
| `WITH_LIBTBB` | `EXTERNAL` | — |
| RapidJSON | always `EXTERNAL` | — |

On **Windows**, most libraries use prebuilt binaries from `win/3rdparty/` (DLLs copied to output directory at build time). Flex/Bison on Windows uses `win_flex_bison` downloaded from the CUBRID mirror.

Static linking is the default on Linux (`--enable-static --disable-shared --with-pic`). Only `libodbc` is built as a shared library (`.so`).

Ninja is fully supported: the `ADD_BY_PRODUCTS_VARIABLE` macro emits `BUILD_BYPRODUCTS` entries so Ninja's dependency graph stays consistent.

## Bundled Libraries

| Library | Version | License | Notes |
|---------|---------|---------|-------|
| [[dependencies/libexpat\|libexpat]] | 2.6.4 (tarball); 2.2.5 listed in README | MIT | XML parsing |
| [[dependencies/jansson\|Jansson]] | 2.10 | MIT | JSON encoding/decoding |
| [[dependencies/libedit\|libedit (Editline)]] | csql_v1.2 (CUBRID fork) | BSD-3 | CLI line editing in csql (Linux only) |
| [[dependencies/openssl\|OpenSSL]] | 1.1.1w (tarball); 1.1.1f in README | OpenSSL/SSLeay | TLS, crypto, SSL connections |
| [[dependencies/unixodbc\|unixODBC]] | 2.3.9 | LGPL-2.1 | ODBC driver manager |
| [[dependencies/lz4\|LZ4]] | 1.9.4 (tarball); 1.9.2 in README | BSD-2 | Fast block compression |
| [[dependencies/rapidjson\|RapidJSON]] | 1.1.0 | MIT | Header-only JSON parser |
| [[dependencies/re2\|RE2]] | 2022-06-01 snapshot | BSD-3 | Regular expression engine |
| [[dependencies/libtbb\|Intel oneTBB]] | 2021.11.0 | Apache 2.0 | Parallel algorithms (server-only) |
| [[dependencies/flex-bison\|Flex / Bison]] | flex 2.6.4 / bison 3.4.1 (Win: winflexbison 2.5.22) | GPL/LGPL | Lexer + parser generator (build-time only) |

> [!note] Separate vendored allocator
> `src/heaplayers/` (Heap Layers by Emery Berger) is a **separate** vendored copy, not managed by this directory. See [[components/heaplayers]].

## Download / Build vs Vendored

- **Linux**: all libraries are downloaded and compiled from source at CMake configure time. No source tarballs are checked into the CUBRID main repo — only `CMakeLists.txt` with verified SHA-256 hashes.
- **Windows**: most libraries are prebuilt binaries checked into `win/3rdparty/` (`.lib` + `.dll`). Only Flex/Bison and RapidJSON download from the mirror.
- **Version drift**: README lists slightly older versions than the actual tarball URLs (e.g., expat 2.2.5 vs 2.6.4, LZ4 1.9.2 vs 1.9.4). Tarball SHA-256 hashes are authoritative.

## Related

- [[dependencies/_index|Dependencies]] — per-library detail pages
- [[Tech Stack]] — languages, build system, CI
- [[Dependency Graph]] — internal + external dep topology
- [[components/heaplayers|heaplayers]] — the other bundled 3rd-party lib (separate from this dir)
- Source: [[sources/cubrid-3rdparty|cubrid-3rdparty]]
