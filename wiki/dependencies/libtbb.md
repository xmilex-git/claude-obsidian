---
type: dependency
name: "Intel oneTBB"
version: "2021.11.0"
source: "https://github.com/CUBRID/3rdparty/raw/develop/tbb/v2021.11.0.tar.gz"
license: "Apache-2.0"
bundled: true
used_by:
  - "Parallel query execution (server-side only)"
  - "Concurrent data structures for parallel scan / aggregation"
risk: low
tags:
  - dependency
  - cubrid
  - parallelism
  - server-only
created: 2026-04-23
updated: 2026-04-23
---

# Intel oneTBB

## What it does

Intel oneAPI Threading Building Blocks (oneTBB) is a C++ template library for task-based parallelism. It provides parallel algorithms (`parallel_for`, `parallel_reduce`), concurrent containers, a thread-local storage API, and a scalable allocator.

## Why CUBRID uses it

CUBRID uses TBB to parallelize query execution — parallel scans and aggregations on the server side. The `"parallel-query"` named thread pool described in the [[components/query-executor|query executor]] uses TBB primitives.

## Integration points

- CMake target: `libtbb`
- **Linux only** — no Windows build in the CMake script (`if(UNIX)` guard)
- Built from source via `ExternalProject_Add` using CMake (not autoconf)
- Key CMake configure flags:
  - `-DBUILD_SHARED_LIBS=OFF` (static)
  - `-DTBBMALLOC_BUILD=OFF -DTBBMALLOC_PROXY_BUILD=OFF` (disables TBB scalable allocator — CUBRID uses its own)
  - `-DTBB_TEST=OFF` (no tests)
  - `-D__TBB_DYNAMIC_LOAD_ENABLED=0` (no dynamic loading of TBB modules)
  - `-DCMAKE_CXX_FLAGS=-DTBB_ALLOCATOR_TRAITS_BROKEN` (compatibility workaround)
- Build: `make -j`; Install: `make -j install`
- **Server-only linking**: TBB is exported as a separate `TBB_TARGETS`/`TBB_LIBS`/`TBB_INCLUDES` set (not in `EP_*`), linked only to `libcubrid.so`, not to client libs

## Risk / notes

- Apache-2.0 — compatible with CUBRID's Apache 2.0 license.
- TBB allocator is intentionally disabled (`TBBMALLOC_BUILD=OFF`); CUBRID manages its own allocation via [[components/heaplayers|Heap Layers]] / `db_private_alloc`.
- TBB is the only 3rdparty library with a separate CMake variable set (`TBB_*` vs `EP_*`) due to its server-only requirement. This design is documented with a `FIXME` comment in `CMakeLists.txt` suggesting the pattern should be generalized.

## Related

- [[components/heaplayers|heaplayers]] — CUBRID's own server-side allocator (TBB allocator disabled to avoid conflict)
- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
