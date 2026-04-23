---
type: dependency
name: "Jansson"
version: "2.10"
source: "https://github.com/CUBRID/3rdparty/raw/develop/jansson/jansson-2.10.tar.gz"
license: "MIT"
bundled: true
used_by:
  - "JSON type support in SQL engine"
  - "CUBRID Manager protocol"
risk: low
tags:
  - dependency
  - cubrid
  - json
created: 2026-04-23
updated: 2026-04-23
---

# Jansson

## What it does

Jansson is a C library for encoding, decoding, and manipulating JSON data. It provides a DOM-style API with reference counting.

## Why CUBRID uses it

CUBRID's native JSON SQL type (`DB_TYPE_JSON=40`) relies on Jansson for JSON document storage and manipulation within the engine. It complements [[dependencies/rapidjson|RapidJSON]], which handles high-performance parsing; Jansson handles the mutable document model.

## Integration points

- CMake target: `libjansson`
- Linux: built from source via `ExternalProject_Add`; produces `libjansson.a` (static)
- Windows: prebuilt `jansson64.dll`/`jansson64.lib` copied from `win/3rdparty/`; two DLL aliases copied (`jansson.dll` and `jansson64.dll`)
- Exposes `LIBJANSSON_LIBS` and `LIBJANSSON_INCLUDES` to parent CMake scope
- Included in `EP_TARGETS`, `EP_LIBS`, `EP_INCLUDES` (all targets)

## Risk / notes

- v2.10 is an older release (2017); upstream is at 2.14.x as of 2024. No known CVEs in 2.10 affecting CUBRID's usage pattern, but drift is worth tracking.
- MIT licensed — no copyleft concerns.
- `DB_TYPE_JSON=40` is ABI-frozen on disk and in the XASL stream — see [[Memory Management Conventions]].

## Related

- [[dependencies/rapidjson|RapidJSON]] — complementary JSON library (header-only, high-performance parsing)
- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
