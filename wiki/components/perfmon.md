---
type: component
parent_module: "[[components/monitor|monitor]]"
path: "src/monitor/"
status: developing
purpose: "Core cubmonitor primitive and composite API — the statistic templates, atomic variants, name-building helpers, and autotimer RAII guards used by all engine instrumentation"
key_files:
  - "monitor_definition.hpp — statistic_value, clock_type, fetch_mode"
  - "monitor_statistic.hpp — primitive<Rep>, atomic_primitive<Rep>, accumulator/gauge/max/min templates"
  - "monitor_collect.hpp — timer_statistic, counter_timer_statistic, counter_timer_max_statistic, build_name_vector"
tags:
  - component
  - cubrid
  - monitor
  - performance
  - statistics
related:
  - "[[components/monitor|monitor]]"
  - "[[components/stats-collection|stats-collection]]"
created: 2026-04-23
updated: 2026-04-23
---

# `perfmon` — Core Statistics Primitives & Composite API

This page covers the collection-side API exported by `src/monitor/`. It is a sub-page of [[components/monitor|monitor]]; read that page first for the full architecture.

## Primitive Templates (`monitor_statistic.hpp`)

All statistics inherit from one of two base primitives:

```
primitive<Rep>          — not thread-safe; for single-writer stats
atomic_primitive<Rep>   — uses std::atomic<Rep>; safe for multi-writer hot paths
```

Four collector policies are layered on top:

```cpp
accumulator_statistic<Rep>        // m_value += change
gauge_statistic<Rep>              // m_value = change   (last write wins)
max_statistic<Rep>                // m_value = max(m_value, change)
min_statistic<Rep>                // m_value = min(m_value, change)
```

Atomic variants use `compare_exchange_strong` loops for `max` and `min`; `fetch_add` for `accumulator`.

### Named Aliases (most commonly used)

```cpp
// Non-atomic (single-writer)
amount_accumulator_statistic      // uint64 counter
time_accumulator_statistic        // duration accumulator (elapsed time)
amount_gauge_statistic            // uint64 current-value gauge
time_max_statistic                // peak duration

// Atomic (shared across threads)
amount_accumulator_atomic_statistic
time_accumulator_atomic_statistic
amount_max_atomic_statistic
time_max_atomic_statistic
```

`floating_rep` (double) atomic variants are disabled by default (`MONITOR_ENABLE_ATOMIC_FLOATING_REP` not defined).

## Composite Statistics (`monitor_collect.hpp`)

### `timer_statistic<T>`

Single-field time accumulator with embedded `timer` helper and RAII `autotimer`:

```cpp
timer_statistic<time_accumulator_atomic_statistic> my_stat;

// RAII: resets timer on entry, calls time() on exit
{
  timer_statistic<...>::autotimer at(my_stat);
  // ... timed work ...
}
```

Aliases: `timer_stat`, `atomic_timer_stat`, `transaction_timer_stat`, `transaction_atomic_timer_stat`.

### `counter_timer_statistic<A, T>`

Pairs a count accumulator with a time accumulator. Registers **3 names** via `register_to_monitor()`:

| Name | Content |
|------|---------|
| `Num_<basename>` | event count |
| `Total_time_<basename>` | cumulative elapsed time |
| `Avg_time_<basename>` | computed at fetch (total / count) — not stored |

RAII `autotimer`: calls `time_and_increment()` on destruction.

Aliases: `counter_timer_stat`, `atomic_counter_timer_stat`.

### `counter_timer_max_statistic<A, T, M>`

Adds a `max` tracker to `counter_timer_statistic`. Registers **4 names**:

| Name | Content |
|------|---------|
| `Num_<basename>` | event count |
| `Total_time_<basename>` | cumulative elapsed time |
| `Max_time_<basename>` | longest single duration |
| `Avg_time_<basename>` | computed at fetch |

> [!key-insight] Max is computed per batch, not per event
> In `time_and_increment(duration d, amount a)`, the max candidate is `d / a` (time per unit, not total). This makes the max meaningful when `a > 1`.

### `build_name_vector()`

Variadic template that prepends each prefix to a basename:

```cpp
std::vector<std::string> names;
build_name_vector(names, "btree_insert", "Num_", "Total_time_", "Avg_time_");
// → ["Num_btree_insert", "Total_time_btree_insert", "Avg_time_btree_insert"]
```

## `fetch_mode`

`fetch_mode` is a `bool` constant:
- `FETCH_GLOBAL = true` — returns the global accumulated value
- `FETCH_TRANSACTION_SHEET = false` — returns the per-transaction delta (see [[components/stats-collection|stats-collection]])

Non-transactional primitives ignore `FETCH_TRANSACTION_SHEET` and return without writing to the destination buffer (destination is left unchanged, typically zero from `allocate_statistics_buffer()`).

## Related

- [[components/monitor|monitor]] — hub: registry, transaction sheets, VACUUM threshold
- [[components/stats-collection|stats-collection]] — per-session vs global aggregation
