---
type: dependency
name: "Flex / Bison"
version: "flex 2.6.4 / bison 3.4.1 (Linux); winflexbison 2.5.22 (Windows)"
source: "https://github.com/CUBRID/3rdparty/raw/develop/flex/ and /bison/"
license: "GPL-2.0 (Bison runtime exception applies to generated code) / LGPL (Flex)"
bundled: false
used_by:
  - "SQL grammar code generation (csql_grammar.y → csql_grammar.cpp)"
  - "SQL lexer code generation (csql_lexer.l → csql_lexer.cpp)"
  - "loaddb grammar"
risk: low
tags:
  - dependency
  - cubrid
  - build-tool
  - parser
  - codegen
created: 2026-04-23
updated: 2026-04-23
---

# Flex / Bison

## What it does

Flex is a lexer generator. Bison is an LALR(1) parser generator. Together they compile grammar specification files (`.l`, `.y`) into C/C++ source code during the build process.

## Why CUBRID uses it

CUBRID's SQL parser is written as a Bison grammar (`src/parser/csql_grammar.y`, 646 KB) with a Flex lexer (`src/parser/csql_lexer.l`). The `loaddb` bulk loader has a separate grammar. These `.y`/`.l` files are compiled at build time into C++ source by Bison and Flex respectively.

## Integration points

- **Build-time tool only** — not linked into CUBRID binaries
- Linux: uses system Flex/Bison (`WITH_LIBFLEXBISON = SYSTEM` by default); `find_package(FLEX 2.5.34 REQUIRED)` and `find_package(BISON 3.0.0 REQUIRED)` enforce minimum versions
- Windows: downloads `win_flex_bison-2.5.22.zip` from the CUBRID mirror; extracts to `win/3rdparty/Install/win_flex_bison/`; sets `FLEX_EXECUTABLE` and `BISON_EXECUTABLE` CMake variables; copies `FlexLexer.h` to `win/3rdparty/include/`
- Building Flex/Bison from source URL on Linux is explicitly unsupported (`FATAL_ERROR`)
- `BISON_ROOT_DIR`, `FLEX_ROOT_DIR` propagated to parent scope on Windows

## Risk / notes

- Bison's GPL applies to the generator itself, not the generated parser code. The Bison Runtime Exception (in bison 2.2+) explicitly exempts generated parsers from GPL copyleft — generated `.cpp` files are Apache 2.0 compatible.
- Flex generates code under its own permissive license (not GPL-encumbered).
- Minimum version constraints: Flex ≥ 2.5.34, Bison ≥ 3.0.0. Build will fail with `FATAL_ERROR` if not met.

## Related

- [[components/parser|parser]] — primary consumer of Flex/Bison generated code
- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
