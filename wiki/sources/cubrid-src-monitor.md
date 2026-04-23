---
type: source
title: "cubrid src/monitor/ — Performance Statistics"
source_path: "src/monitor/"
date_ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - monitor
  - performance
  - statistics
created_pages:
  - "[[components/monitor]]"
  - "[[components/perfmon]]"
  - "[[components/stats-collection]]"
---

# Source: `src/monitor/`

Ingest of the CUBRID performance-statistics subsystem (`src/monitor/`). Read files: `AGENTS.md`, all six `.hpp` headers, `monitor_registration.cpp`, `monitor_transaction.cpp`.

## Files Covered

| File | Lines (approx) | Notes |
|------|---------------|-------|
| `monitor_definition.hpp` | 47 | Wire type + clock aliases |
| `monitor_statistic.hpp` | 595 | All primitive + collector templates |
| `monitor_transaction.hpp` | 276 | Sheet manager + `transaction_statistic<S>` |
| `monitor_collect.hpp` | 567 | Composite stats + `autotimer` RAII |
| `monitor_registration.hpp` | 147 | `monitor` class interface |
| `monitor_registration.cpp` | 164 | Global singleton, fetch loop |
| `monitor_transaction.cpp` | 203 | Sheet manager static-init, start/end/get |
| `monitor_vacuum_ovfp_threshold.hpp` | 140 | Server-only ovfp types |
| `monitor_vacuum_ovfp_threshold.cpp` | (not read) | Implementation |

## Key Facts

- `statistic_value = uint64_t` is the universal wire type; internal reps (`amount_rep`, `floating_rep`, `time_rep`) are cast at fetch.
- Four policies: accumulator, gauge, max, min — each with atomic and non-atomic variants.
- `time_rep` atomics use `std::atomic<duration::rep>` (tick count) because `std::atomic<duration>` is invalid.
- Always-on: no sampling gate; overhead is bounded by the atomic primitive chosen.
- Per-transaction sheets: reused (not zeroed); callers must snapshot-delta.
- Sheet capacity: max 1024 concurrent (`MAX_SHEETS`); sheet array per `transaction_statistic<S>` grows lazily.
- Global monitor is a file-static `monitor Monitor` in `monitor_registration.cpp`.
- `counter_timer_statistic::register_to_monitor()` auto-generates `Num_`, `Total_time_`, `Avg_time_` names.
- VACUUM ovfp threshold monitor is server-only, uses its own mutex locking, and operates on `BTID`/`OID` domain objects.

## Pages Created

- [[components/monitor]] — component hub: full architecture, wire type, build integration, public API
- [[components/perfmon]] — primitive and composite API detail: template grid, aliases, autotimer patterns
- [[components/stats-collection]] — aggregation model: always-on globals, sheet lifecycle, snapshot-delta, overhead table
