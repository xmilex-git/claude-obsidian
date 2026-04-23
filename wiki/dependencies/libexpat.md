---
type: dependency
name: "libexpat"
version: "2.6.4 (tarball SHA256: fd03b7172…); README says 2.2.5"
source: "https://github.com/CUBRID/3rdparty/raw/develop/expat/expat-2.6.4.tar.gz"
license: "MIT"
bundled: true
used_by:
  - "XML config / protocol parsing inside the engine"
risk: low
tags:
  - dependency
  - cubrid
  - xml
created: 2026-04-23
updated: 2026-04-23
---

# libexpat

## What it does

libexpat is a stream-oriented XML parser library written in C. It parses XML incrementally without building a DOM tree.

## Why CUBRID uses it

CUBRID uses libexpat for XML document processing, primarily in configuration file parsing and XML-type data handling in the SQL engine.

## Integration points

- CMake target: `libexpat`
- Linux: built from source via `ExternalProject_Add`; produces `libexpat.a` (static)
- Windows: prebuilt `libexpat64.dll` copied from `win/3rdparty/`; DLL renamed to `libexpat.dll` at build time
- Exposes `LIBEXPAT_LIBS` and `LIBEXPAT_INCLUDES` into the parent CMake scope via `expose_3rdparty_variable(LIBEXPAT)`
- Configure flags: `--without-xmlwf --without-docbook` (strips CLI tool and docs)

## Risk / notes

- Version in README (2.2.5) differs from actual tarball URL (2.6.4). The tarball + SHA-256 hash is authoritative.
- MIT licensed — no copyleft concerns.
- Static on Linux; DLL on Windows (ABI compatibility dependency).

## Related

- [[modules/3rdparty|3rdparty module]] — build orchestration
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
