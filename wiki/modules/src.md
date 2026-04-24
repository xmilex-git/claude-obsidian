---
type: module
path: "src/"
status: active
language: "C/C++17"
purpose: "CUBRID database engine — all server-side and client-side database logic"
last_updated: 2026-04-23
depends_on:
  - "[[modules/3rdparty|3rdparty]]"
  - "[[modules/cm_common|cm_common]]"
used_by:
  - "[[modules/cubrid|cubrid (server binary)]]"
  - "[[modules/sa|sa]]"
  - "[[modules/cs|cs]]"
tags:
  - module
  - cubrid
  - core
related:
  - "[[Architecture Overview]]"
  - "[[Query Processing Pipeline]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/` — Database Engine Source

The heart of CUBRID. C/C++17. The same source compiles into three different binaries via [[Build Modes (SERVER SA CS)|SERVER_MODE / SA_MODE / CS_MODE]] guards.

## Subdirectory map

| Path | Purpose | Component page |
|------|---------|----------------|
| `parser/` | SQL → parse tree → XASL (bison/flex) | [[components/parser]] |
| `optimizer/` | Cost-based query planning | [[components/optimizer]] |
| `query/` | XASL execution, scan managers | [[components/query]] |
| `storage/` | Buffer pool, heap files, B-tree | [[components/storage]] |
| `transaction/` | MVCC, WAL, locking, recovery, boot | [[components/transaction]] |
| `object/` | Schema, auth, information_schema | [[components/object]] |
| `compat/` | Client API (`db_*`), `DB_VALUE` | [[components/compat]] |
| `base/` | Error handling, memory, lock-free, porting | [[components/base]] |
| `xasl/` | XASL node type definitions | [[components/xasl]] |
| `executables/` | Binaries: `cub_server`, `csql`, utilities | [[components/executables]] |
| `broker/` | Connection broker (CAS processes) | [[components/broker-impl]] |
| `sp/` | Stored procedure JNI bridge | [[components/sp]] |
| `connection/` | Client-server TCP / heartbeat | [[components/connection]] |
| `method/` | Method/SP invocation from queries | [[components/method]] |
| `thread/` | Worker pools, daemons (C++17) | [[components/thread]] |
| `loaddb/` | Bulk loader (bison/flex grammar) | [[components/loaddb]] |
| `monitor/` | Performance statistics | [[components/monitor]] |
| `session/` | Per-connection session state | [[components/session]] |
| `communication/` | Internal protocol (C++) | [[components/communication]] |
| `heaplayers/` | Embedded malloc/heap allocators (3rd-party) | [[components/heaplayers]] |
| `cm_common/` | CUBRID Manager shared utils | [[components/cm-common-src]] |
| `api/` | Public C API extensions (`cubrid_log.c`) | [[components/api]] |
| `debugging/` | Compiler warning helpers, type utilities | [[components/debugging]] |
| `win_tools/` | Windows service / tray tools | [[components/win-tools]] |

## Critical gotchas

> [!warning] Two `broker/` directories
> Top-level [[modules/broker|broker/]] (CMake target + configs) ≠ `src/broker/` (actual implementation). Don't confuse them.

> [!warning] `csql_grammar.y` is 646 KB
> Bison grammar in `parser/`. Edits need extreme care; regeneration is slow.

> [!info] Large files are intentional
> Several files exceed 10 K lines (some 30 K+). Per project policy, **do not split them** — the size is intentional, not tech debt.

## Sub-module guides

`src/AGENTS.md` exists with deeper detail per the project guide. Will be ingested separately.

## Related

- Source: [[cubrid-AGENTS]]
- Hubs: [[Architecture Overview]], [[Tech Stack]], [[Data Flow]]
- Modules index: [[modules/_index]]
