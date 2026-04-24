---
status: reference
type: dependency
name: "LZ4"
version: "1.9.4 (tarball); README lists 1.9.2"
source: "https://github.com/CUBRID/3rdparty/raw/develop/lz4/v1.9.4.tar.gz"
license: "BSD-2-Clause"
bundled: true
used_by:
  - "Page-level compression in the storage engine"
  - "WAL log compression"
risk: low
tags:
  - dependency
  - cubrid
  - compression
  - storage
created: 2026-04-23
updated: 2026-04-23
---

# LZ4

## What it does

LZ4 is an extremely fast lossless compression algorithm, optimized for speed over ratio. It provides both standard LZ4 frame format and the LZ4-HC (high-compression) variant.

## Why CUBRID uses it

CUBRID uses LZ4 for compressing data pages and WAL log records, trading a small CPU overhead for reduced I/O bandwidth and storage footprint at the page/record level.

## Integration points

- CMake target: `lz4`
- Linux: built from source via `ExternalProject_Add` using the LZ4 Makefile directly (`BUILD_IN_SOURCE true`)
- No `configure` step — LZ4 uses a simple Makefile; builds `liblz4.a` in `Source/lz4/lib/`
- Build flag: `CFLAGS=-fPIC` (required for static linking into shared libraries like `libodbc.so`)
- Windows: prebuilt `liblz4.dll`/`liblz4.lib` from `win/3rdparty/lz4/`; DLL copied to runtime output directory
- Exposes `LZ4_LIBS` and `LZ4_INCLUDES` via `expose_3rdparty_variable(LZ4)`
- Included in `EP_TARGETS`, `EP_LIBS`, `EP_INCLUDES` (all targets)

## Risk / notes

- Version drift: README says 1.9.2; actual tarball is 1.9.4. Positive drift.
- BSD-2-Clause — permissive, no concerns.
- `BUILD_IN_SOURCE true` means the build directory is the source directory — clean rebuilds require clearing the ExternalProject source.

## Related

- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
