---
type: component
parent_module: "[[components/monitor|monitor]]"
path: "src/monitor/"
status: developing
purpose: "Counter aggregation model: always-on global accumulators + optional per-transaction sheet isolation; snapshot-delta pattern for transaction-scoped perf analysis"
key_files:
  - "monitor_transaction.hpp/cpp — transaction_sheet_manager, transaction_statistic<S>"
  - "monitor_registration.hpp/cpp — fetch_global_statistics(), fetch_transaction_statistics()"
tags:
  - component
  - cubrid
  - monitor
  - statistics
  - transaction
related:
  - "[[components/monitor|monitor]]"
  - "[[components/perfmon|perfmon]]"
created: 2026-04-23
updated: 2026-04-23
---

# `stats-collection` — Counter Aggregation Per Session / Global

This page covers the dual-scope aggregation model in `src/monitor/`. It is a sub-page of [[components/monitor|monitor]].

## Two Scopes

Every `cubmonitor` statistic can operate in two scopes simultaneously:

| Scope | Storage | Who updates | When fetched |
|-------|---------|-------------|--------------|
| **Global** | `m_global_stat` in `transaction_statistic<S>`, or the bare primitive | All threads, always | `FETCH_GLOBAL` |
| **Transaction sheet** | `m_sheet_stats[sheet_id]` | Only threads whose transaction is being watched | `FETCH_TRANSACTION_SHEET` |

Global stats are **always-on** — there is no overhead gate. Sheet stats only accumulate when a sheet is active (i.e., `get_sheet()` returns a valid index).

## Transaction Sheet Lifecycle

```
start_watch()          → allocate sheet slot, refcount = 1 (re-entrant: refcount++)
   │
   │   collect() calls:
   │     global_stat.collect(v)           ← always
   │     sheet_stats[sheet_id].collect(v) ← if sheet active
   │
   ▼
end_watch()            → refcount--; if refcount == 0 → release slot
```

`transaction_sheet_manager` is an entirely static class (no instances). Sheet slots are integers (0–1023) that map to transaction indices via `s_transaction_sheets[tran_index - 1]`.

### Why Sheets Are Reused

Allocating a new `statistic_type[]` array per transaction would be too expensive. Instead, slots are recycled. The sheet's counters are **not zeroed** on reuse — this is intentional. The caller must:

1. Call `allocate_statistics_buffer()` + `fetch_transaction_statistics()` at `start_watch()` → **baseline snapshot**.
2. Call `fetch_transaction_statistics()` again at `end_watch()` → **end snapshot**.
3. Compute `end - baseline` per field.

> [!warning] Never read a sheet value without a baseline delta
> A non-zero value in a freshly assigned sheet is a leftover from the previous tenant. Reading it raw gives meaningless results.

### Sheet Capacity

- Maximum 1024 concurrent sheets (`MAX_SHEETS`).
- Sheet array grows lazily: `transaction_statistic<S>::extend()` allocates a new array (copy-on-grow, protected by `m_extend_mutex`) when a higher-numbered sheet is seen for the first time.
- `start_watch()` returns `false` if all 1024 slots are taken (rare: each open stat-watch uses one slot).

### Daemon / Vacuum Thread Behaviour

`get_sheet()` checks `logtb_get_current_tran_index()`. System transaction index (`LOG_SYSTEM_TRAN_INDEX`) and out-of-range indices (daemon threads, vacuum workers) return `INVALID_TRANSACTION_SHEET`, so sheet collection is silently skipped — daemons only contribute to global counters.

## Global Fetch Path

```cpp
// monitor::fetch_global_statistics(buf):
for (auto &reg : m_registrations)
  reg.m_fetch_func(buf_ptr, FETCH_GLOBAL);
  buf_ptr += reg.m_statistics_count;
```

The buffer is a flat `uint64_t[]` aligned to the registration order. `statdump` reads this buffer, matches names from the name table (`m_all_names`), and formats output.

## Transaction Fetch Path

```cpp
// monitor::fetch_transaction_statistics(buf):
if (get_sheet() == INVALID_TRANSACTION_SHEET) return; // nothing to do
fetch_statistics(buf, FETCH_TRANSACTION_SHEET);
```

Each `transaction_statistic<S>::fetch()` with `FETCH_TRANSACTION_SHEET` looks up `get_sheet()` and reads from `m_sheet_stats[sheet]` (or returns zero if the array is not yet extended to that index).

## Overhead Characteristics

| Operation | Cost |
|-----------|------|
| Global collect (`atomic_accumulator`) | 1 `fetch_add` |
| Sheet collect (sheet active) | 1 `get_sheet()` lookup + 1 non-atomic add |
| Sheet collect (no sheet) | 1 `get_sheet()` lookup (branch taken, skip) |
| Global fetch (N stats) | N function calls across registered closures |
| No sampling gate | All stats collected unconditionally — no `if (enabled)` check |

Statistics are **always-on**: there is no compile-time or runtime switch to disable the `cubmonitor` layer globally. The only overhead control is choosing atomic vs non-atomic primitives for each stat.

## Related

- [[components/monitor|monitor]] — hub: registry and VACUUM threshold
- [[components/perfmon|perfmon]] — primitive templates and composite stat classes
