---
status: reference
type: dependency
name: "libedit (Editline)"
version: "csql_v1.2 (CUBRID fork of libedit)"
source: "https://github.com/CUBRID/libedit/archive/refs/tags/csql_v1.2.tar.gz"
license: "BSD-3-Clause"
bundled: true
used_by:
  - "csql interactive shell (line editing, history, completion)"
risk: low
tags:
  - dependency
  - cubrid
  - cli
created: 2026-04-23
updated: 2026-04-23
---

# libedit (Editline)

## What it does

libedit (also known as editline) is a BSD-licensed line-editing library providing readline-compatible functionality: command history, cursor movement, completion callbacks.

## Why CUBRID uses it

CUBRID's interactive SQL shell (`csql`) uses libedit for command-line editing and history. It provides a readline-compatible API without GPL licensing concerns.

## Integration points

- CMake target: `libedit` (Linux only — `if(UNIX)` guard)
- Built from source via `ExternalProject_Add`; produces `libedit.a` (static)
- Configure flag: `LDFLAGS=-L${3RDPARTY_LIBS_DIR}` to pick up bundled ncurses if present; `LIBS=-ldl`
- Depends on `LIBNCURSES_TARGET` if it exists (ncurses is a libedit runtime dependency)
- Exposes `LIBEDIT_LIBS` and `LIBEDIT_INCLUDES` via `expose_3rdparty_variable(LIBEDIT)`
- **Not built on Windows** — csql on Windows uses a different line-editing mechanism

## Risk / notes

- This is a **CUBRID fork** (`github.com/CUBRID/libedit`), not upstream libedit. The fork is tagged `csql_v1.2`; fork divergence from upstream should be reviewed periodically.
- BSD-3-Clause licensed — no copyleft concerns.

## Related

- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
