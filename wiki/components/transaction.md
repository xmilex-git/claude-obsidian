---
type: component
parent_module: "[[modules/src|src]]"
path: "src/transaction/"
status: active
purpose: "MVCC, WAL, locking, recovery, boot"
key_files:
  - "mvcc.c (mvcc_satisfies_snapshot)"
  - "mvcc.h (MVCC_TRANS_STATUS struct)"
  - "lock_manager.c"
  - "lock_manager.h (LOCK_RESOURCE struct)"
  - "wait_for_graph.c (deadlock detection)"
  - "log_*.c (WAL)"
  - "log_append.cpp (write path)"
  - "log_record.hpp (LOG_RECORD_HEADER struct)"
public_api: []
tags:
  - component
  - cubrid
  - transaction
  - mvcc
  - wal
  - server
related:
  - "[[modules/src|src]]"
  - "[[components/storage|storage]]"
  - "[[Architecture Overview]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/transaction/` — Transactions, MVCC, WAL, Locking

Concurrency control + durability + recovery for the CUBRID engine.

## Sub-areas

| Area | Files | Notes |
|------|-------|-------|
| MVCC visibility | `mvcc.c`, `mvcc.h` | Entry: `mvcc_satisfies_snapshot()` — uses `MVCC_TRANS_STATUS` snapshots |
| Lock manager | `lock_manager.c`, `lock_manager.h` | `LOCK_RESOURCE` = lock + owner list + waiters |
| Deadlock detection | `wait_for_graph.c` | Wait-for graph algorithm |
| Write-ahead log | `log_*.c`, `log_append.cpp` | WAL: writes go through `log_append.cpp`; record header `LOG_RECORD_HEADER` (with LSN, txn ID) lives in `log_record.hpp` |
| Boot / recovery | (boot_*, recovery_*) | Crash recovery on startup |

## Common modifications (from [[cubrid-AGENTS|AGENTS.md]])

- **Fix MVCC visibility** → `mvcc.c`, function `mvcc_satisfies_snapshot()`
- **Fix locking / deadlock** → `lock_manager.c` + `wait_for_graph.c`
- **Fix WAL / recovery** → `log_*.c`; writes via `log_append.cpp`

## Server-side only

Most of this is `SERVER_MODE` code. Standalone (`SA_MODE`) hosts the same logic in-process.

## Related

- Parent: [[modules/src|src]]
- [[components/storage]] (the things being protected)
- Source: [[cubrid-AGENTS]]
