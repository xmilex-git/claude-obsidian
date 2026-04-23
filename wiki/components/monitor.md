---
type: component
parent_module: "[[modules/src|src]]"
path: "src/monitor/"
status: developing
purpose: "Runtime performance statistics collection and export — accumulator/gauge/max/min primitives, per-transaction sheet tracking, global named registry, and a server-only VACUUM overflow-page threshold monitor"
key_files:
  - "monitor_definition.hpp — statistic_value (uint64), clock aliases, fetch_mode constants"
  - "monitor_statistic.hpp/cpp — primitive<Rep>, atomic_primitive<Rep>, accumulator/gauge/max/min collector templates"
  - "monitor_transaction.hpp/cpp — transaction_sheet_manager, transaction_statistic<S> wrapper"
  - "monitor_collect.hpp/cpp — timer_statistic, counter_timer_statistic, counter_timer_max_statistic, name builders"
  - "monitor_registration.hpp/cpp — monitor class: register_statistics(), fetch_global_statistics(), get_global_monitor()"
  - "monitor_vacuum_ovfp_threshold.hpp/cpp — server-only ovfp_threshold_mgr (VACUUM overflow-page threshold)"
tags:
  - component
  - cubrid
  - monitor
  - performance
  - statistics
related:
  - "[[components/system-catalog|system-catalog]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/btree|btree]]"
  - "[[components/lock-manager|lock-manager]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[components/lockfree|lockfree]]"
  - "[[components/vacuum|vacuum]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[components/perfmon|perfmon]]"
  - "[[components/stats-collection|stats-collection]]"
created: 2026-04-23
updated: 2026-04-23
---

# `monitor` — Runtime Performance Statistics

`src/monitor/` is CUBRID's centralized performance-statistics layer (namespace `cubmonitor`). It provides typed statistic primitives, a per-transaction dual-write mechanism, a global named registry, and a server-only VACUUM overflow-page threshold monitor. The `statdump` CLI tool and any future monitoring views all consume this layer.

## Architecture Overview

```
Producer (engine thread)
  │
  │  .collect(value)
  ▼
transaction_statistic<S>          ← optional wrapper
  ├── m_global_stat  [always written]
  └── m_sheet_stats[sheet_id]     ← written only when sheet is active
           │
           │  .fetch(buf, mode)
           ▼
     monitor  (global registry)
       ├── registration[0]  → fetch_function → statistic_value[]
       ├── registration[1]  → fetch_function → statistic_value[]
       └── ...
           │
           ▼
     statdump / system catalog
```

## Wire Type: `statistic_value`

Every stat is exported as `std::uint64_t` (`statistic_value`). Internally, three representations are used and cast at fetch time:

| Alias | Underlying C++ type | Use |
|-------|---------------------|-----|
| `amount_rep` | `uint64_t` | Counters, row counts |
| `floating_rep` | `double` | Ratios, percentages |
| `time_rep` | `std::chrono::duration` | Elapsed time |

Clocking uses `std::chrono::high_resolution_clock` (`clock_type`).

## Statistic Primitives (`monitor_statistic.hpp`)

Four collector policies, each with a non-atomic and a `std::atomic<Rep>` variant:

| Policy | collect() behaviour | Atomic variant |
|--------|--------------------|---------------|
| `accumulator_statistic<Rep>` | `value += change` | `accumulator_atomic_statistic<Rep>` |
| `gauge_statistic<Rep>` | `value = change` (last write wins) | `gauge_atomic_statistic<Rep>` |
| `max_statistic<Rep>` | CAS loop until `value >= change` | `max_atomic_statistic<Rep>` |
| `min_statistic<Rep>` | CAS loop until `value <= change` | `min_atomic_statistic<Rep>` |

All inherit from `primitive<Rep>` or `atomic_primitive<Rep>`. Both base classes expose the same `fetch(statistic_value*, fetch_mode)` interface used by the registry.

> [!key-insight] Atomic time_rep is specially handled
> `std::atomic<duration>` is not valid. `atomic_primitive<time_rep>` stores `std::atomic<duration::rep>` (the underlying tick count) and reconstructs `duration` on fetch. This avoids a lock for the common case of cross-thread timer accumulation.

Named aliases follow the pattern `<rep>_<policy>[_atomic]_statistic`, e.g.:
- `time_accumulator_atomic_statistic` — timer shared across threads
- `amount_gauge_statistic` — single-writer current-value gauge
- `amount_max_atomic_statistic` — CAS-based high-water mark

`floating_rep` atomic variants are conditionally compiled under `MONITOR_ENABLE_ATOMIC_FLOATING_REP` (not set by default; `fetch_add` for doubles uses a CAS retry loop).

## Per-Transaction Sheets (`monitor_transaction.hpp`)

`transaction_statistic<S>` wraps any statistic type and adds optional per-transaction isolation:

```
transaction_statistic<time_accumulator_atomic_statistic>
  ├── m_global_stat           ← always updated on collect()
  └── m_sheet_stats[]         ← sized on first demand; extended with m_extend_mutex
```

`transaction_sheet_manager` (fully static class) manages the mapping `tran_index → sheet_id`:

| Method | Effect |
|--------|--------|
| `start_watch()` | Allocate a sheet slot for the current transaction (re-entrant: reference-counted) |
| `end_watch(end_all)` | Decrement ref count; release slot when count reaches 0 |
| `get_sheet()` | O(1) lookup via `s_transaction_sheets[tran_index - 1]`; returns `INVALID_TRANSACTION_SHEET` for daemons/vacuum workers |

Key constraint: sheets are **reused** — values are not zeroed on assignment. Callers must snapshot at `start_watch()` and compute the delta on `end_watch()`.

Sheet array is lazy-allocated on first `start_watch()` call, sized to `NUM_NORMAL_TRANS`. Max 1024 concurrent sheets (`MAX_SHEETS`).

## Global Registry (`monitor_registration.hpp/.cpp`)

`monitor` is a singleton (`get_global_monitor()`) that owns a flat list of registrations:

```cpp
struct registration {
  std::size_t m_statistics_count;
  fetch_function m_fetch_func;   // std::function<void(statistic_value*, fetch_mode)>
};
```

`register_statistics(count, fetch_fn, names)` appends one registration and extends the flat name table. At fetch time:

```cpp
// fetch loop in fetch_statistics():
statistic_value *p = destination;
for (auto &reg : m_registrations) {
  reg.m_fetch_func(p, mode);
  p += reg.m_statistics_count;
}
```

The buffer is a flat `uint64_t[]` of `get_statistics_count()` elements — no per-field headers, just sequential values aligned to the name table.

## Composite Statistics (`monitor_collect.hpp`)

Higher-level grouped statistics built on primitives:

### `timer_statistic<T>`
One time accumulator with an embedded `timer` (resets on `reset_timer()`). Inner `autotimer` RAII guard: starts timer on construction, calls `time()` on destruction.

### `counter_timer_statistic<A, T>`
Two statistics: count (`amount_rep` accumulator) + elapsed time. Registers three names when calling `register_to_monitor()`: `Num_<basename>`, `Total_time_<basename>`, `Avg_time_<basename>` (average is computed at fetch, not stored).

### `counter_timer_max_statistic<A, T, M>`
Extends `counter_timer_statistic` with a `max` time tracker. Registers four names: `Num_`, `Total_time_`, `Max_time_`, `Avg_time_`.

`build_name_vector()` is a variadic template helper that builds the `std::vector<std::string>` from a basename and a list of prefix strings.

## VACUUM Overflow-Page Threshold (`monitor_vacuum_ovfp_threshold.hpp`)

Server-only (`#if defined(SERVER_MODE)`). Tracks B-tree overflow-page read counts per vacuum worker to detect indices with chronic overflow.

```
ovfp_threshold_mgr
  ├── m_ovfp_lock (ovfp_monitor_lock)      ← per-worker slot mutex
  └── m_ovfp_threshold[VACUUM_MAX_WORKER_COUNT]
          └── INDEX_OVFP_INFO linked list  ← per (class_oid, btid) pair
                  ├── hit_cnt
                  ├── read_pages[RECENT/MAX]
                  └── event_time[RECENT/MAX]
```

`add_read_pages_count(thread_p, worker_idx, btid, npages)` is the collection call; `dump()` prints sorted results. Unlike the `cubmonitor` namespace statistics, this uses its own mutex-based locking and operates on `BTID`/`OID` domain objects, not raw `statistic_value`.

## Build Integration

| Component | Server | SA | CS |
|-----------|--------|----|----|
| `monitor_statistic`, `monitor_transaction`, `monitor_collect`, `monitor_registration` | Yes | Yes | No |
| `monitor_vacuum_ovfp_threshold` | Yes | No | No |

CS clients do not link any monitor code; they consume exported stat values from the server via the protocol layer.

## Public API Summary

```cpp
// Registry
cubmonitor::monitor &get_global_monitor();
monitor::register_statistics(count, fetch_fn, names);
monitor::allocate_statistics_buffer() -> statistic_value*;
monitor::fetch_global_statistics(buf);
monitor::fetch_transaction_statistics(buf);

// Per-transaction sheets
transaction_sheet_manager::start_watch();
transaction_sheet_manager::end_watch(end_all = false);
transaction_sheet_manager::get_sheet() -> transaction_sheet;

// Composite
counter_timer_statistic<A,T>::register_to_monitor(monitor&, basename);
counter_timer_max_statistic<A,T,M>::register_to_monitor(monitor&, basename);
```

## Usage Pattern for Producers

```cpp
// Declare (often as a global or struct member)
cubmonitor::atomic_counter_timer_stat g_btree_insert_stat;

// In the hot path (lock-free if using atomic variant)
{
  cubmonitor::counter_timer_statistic<...>::autotimer at(g_btree_insert_stat);
  // ... do the insert ...
} // destructor calls time_and_increment()

// At startup — register once
g_btree_insert_stat.register_to_monitor(get_global_monitor(), "btree_insert");
```

## Related

- [[components/perfmon|perfmon]] — `perfmon_*` legacy macro API (wraps `cubmonitor` or the older `perf_stat_` arrays)
- [[components/stats-collection|stats-collection]] — counter aggregation details: session vs global scopes
- [[components/system-catalog|system-catalog]] — exposes statistics as `SHOW STATISTICS` / information-schema views
- [[components/vacuum|vacuum]] — consumer of `ovfp_threshold_mgr`; VACUUM workers call `add_read_pages_count`
- [[components/page-buffer|page-buffer]], [[components/btree|btree]], [[components/lock-manager|lock-manager]] — instrumented producers
- [[components/lockfree|lockfree]] — atomic primitives (CAS, `fetch_add`) underpin the atomic statistic variants
- [[components/system-parameter|system-parameter]] — `PRM_ID_*` toggles may gate statistic collection
- [[Build Modes (SERVER SA CS)]] — CS build excludes all monitor sources
- Source: [[sources/cubrid-src-monitor|cubrid-src-monitor]]
