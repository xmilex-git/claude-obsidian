---
type: component
parent_module: "[[modules/src|src]]"
path: "src/debugging/"
status: developing
purpose: "Minimal header-only debug utilities: compile-time type name resolution and (planned) compiler warning suppression helpers"
key_files:
  - "type_helper.hpp (DBG_REGISTER_PARSE_TYPE_NAME + dbg_parse_type_name<T>, debug-only, #if !defined(NDEBUG))"
tags:
  - component
  - cubrid
  - debugging
  - type-utilities
  - compiler-warnings
related:
  - "[[modules/src|src]]"
  - "[[components/base|base]]"
  - "[[Code Style Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/debugging/` — Compiler & Debug Utilities

A small, header-only directory providing compile-time and debug-build utilities. It has no runtime dependencies and no `.c`/`.cpp` translation units — everything is `constexpr` or preprocessor macros, active only in debug builds (`#if !defined(NDEBUG)`).

## Directory Contents

| File | Purpose | Active when |
|------|---------|-------------|
| `type_helper.hpp` | Compile-time type name stringification | `NDEBUG` not defined (debug builds) |
| `strict_warnings` | Compiler warning suppression helpers (referenced in AGENTS.md; not yet present in tree) | — |

## `type_helper.hpp` — Compile-Time Type Names

### What it does

Provides a mechanism to get a human-readable string name for any C++ type at compile time, useful for printing template instantiation info in debug output.

### API

```cpp
// Step 1: register a type (one specialization per type, at namespace scope)
DBG_REGISTER_PARSE_TYPE_NAME (MyType);

// Step 2: retrieve the name as a constexpr string
constexpr const char *name = dbg_parse_type_name<MyType> ();
```

### Implementation details

- `DBG_REGISTER_PARSE_TYPE_NAME(_TYPE_)` expands to a full template specialization:
  ```cpp
  template <>
  constexpr const char* dbg_parse_type_name<_TYPE_>()
  {
    return MAKE_STRING(_TYPE_);  // stringifies via ## preprocessor operator
  }
  ```
- `MAKE_STRING(x)` is a two-level macro (`MAKE_STRING_IMPL` + `MAKE_STRING`) to force macro expansion before stringification — a standard C++ preprocessor idiom.
- The forward declaration `template <typename T> constexpr const char *dbg_parse_type_name ();` is provided so callers get a link-time error (rather than a silent wrong answer) if they forget to register a type.
- The entire file body is wrapped in `#if !defined(NDEBUG)` — it contributes nothing to release builds, making it zero-overhead in production.
- Header guard: `TYPE_HELPER_HPP` (uses `UPPER_SNAKE` without leading/trailing underscores, consistent with CUBRID style for `.hpp` files).

### Typical use case

Template classes or functions that need to log or assert their exact instantiated type during debugging, without relying on `typeid` / RTTI (which is generally disabled in CUBRID engine builds).

## `strict_warnings` (planned / absent)

The root `AGENTS.md` lists `strict_warnings` as an owned file alongside `type_helper`. No file with that name exists in the tree under any extension (`.h`, `.hpp`). It is either:
- Planned but not yet implemented, or
- Previously removed.

If added, it would presumably contain `#pragma GCC diagnostic` push/pop helpers or equivalent MSVC `__pragma` wrappers for selectively silencing compiler warnings around known-noisy third-party code paths.

## Design Notes

- **Zero runtime cost**: the entire directory produces no object files, no symbols, and no binary size in release builds.
- **No upstream dependencies**: `type_helper.hpp` includes nothing — not even `<type_traits>`. It is safe to include anywhere in the engine.
- **Debug-guard pattern**: wrapping with `#if !defined(NDEBUG)` (rather than a custom `CUBRID_DEBUG` macro) aligns with the standard CMake `CMAKE_BUILD_TYPE=Release` → `-DNDEBUG` convention used by CUBRID's build scripts.

## Related

- Parent: [[modules/src|src]]
- [[components/base|base]] — similar utility-layer nature; error/memory infra used across all components
- [[Code Style Conventions]] — CI-enforced style; `strict_warnings` would interact with CI lint pipeline
- Source: [[sources/cubrid-src-debugging|cubrid-src-debugging]]
