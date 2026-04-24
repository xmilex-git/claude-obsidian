---
status: reference
type: dependency
name: "RapidJSON"
version: "1.1.0"
source: "https://github.com/CUBRID/3rdparty/raw/develop/rapidjson/v1.1.0.tar.gz"
license: "MIT"
bundled: true
used_by:
  - "High-performance JSON parsing for JSON SQL type"
  - "JSON document serialization/deserialization"
risk: low
tags:
  - dependency
  - cubrid
  - json
  - header-only
created: 2026-04-23
updated: 2026-04-23
---

# RapidJSON

## What it does

RapidJSON is a fast, header-only C++ JSON parser/generator. It claims performance comparable to `strlen()` for parsing. It supports SAX and DOM styles, schema validation, and SIMD-accelerated parsing on x86.

## Why CUBRID uses it

RapidJSON provides the high-performance parsing path for CUBRID's JSON SQL type. Being header-only, there is no separate build step — only include directories are needed. It complements [[dependencies/jansson|Jansson]], which handles mutable document manipulation; RapidJSON handles the parsing/generation hot path.

## Integration points

- CMake target: `rapidjson`
- **Header-only**: no library file produced; `RAPIDJSON_LIBS = ""`
- `ExternalProject_Add` downloads the tarball and sets include path to `Source/rapidjson/include`
- CMake options disable tests, docs, and examples: `-DRAPIDJSON_BUILD_TESTS=off -DRAPIDJSON_BUILD_DOC=off -DRAPIDJSON_BUILD_EXAMPLES=off`
- No `INSTALL_COMMAND` — headers remain in the build tree source directory
- Exposes `RAPIDJSON_INCLUDES` via `expose_3rdparty_variable(RAPIDJSON)`
- Included in `EP_TARGETS`, `EP_INCLUDES` (all targets)
- Works on both Linux and Windows (header-only; no platform-specific handling)

## Risk / notes

- v1.1.0 (2016) is quite old; upstream is at 1.1.0 + many patches on `master`. The project has had infrequent releases.
- MIT licensed — no concerns.
- Header-only means no compilation risk, but the upstream API may diverge from what CUBRID's engine expects if ever upgraded.

## Related

- [[dependencies/jansson|Jansson]] — complementary JSON library (mutable DOM, runtime encoding/decoding)
- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
