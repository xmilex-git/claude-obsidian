---
type: component
parent_module: "[[modules/src|src]]"
path: "src/thread/"
status: stable
purpose: "Long-running background daemon threads with configurable sleep/wake patterns; used for vacuum, log flush, page flush, checkpoint, and more"
key_files:
  - "thread_daemon.hpp (cubthread::daemon class)"
  - "thread_looper.hpp (cubthread::looper — wait strategy)"
  - "thread_waiter.hpp (cubthread::waiter — condition variable wrapper)"
  - "thread_manager.hpp (create_daemon / destroy_daemon factory methods)"
public_api:
  - "cubthread::daemon::wakeup() — signal daemon to run immediately"
  - "cubthread::daemon::stop_execution() — stop loop and join thread"
  - "cubthread::daemon::was_woken_up() — true if last wakeup was explicit (not timeout)"
  - "cubthread::daemon::reset_looper() — reset increasing-period counter"
  - "cubthread::daemon::is_running() — true if thread is still alive"
  - "cubthread::daemon::get_stats(stat_value*) — daemon statistics"
  - "cubthread::looper — construct with desired wait pattern"
  - "looper::put_to_sleep(waiter&) — sleep daemon for one period"
  - "looper::stop() / is_stopped() / was_woken_up()"
  - "cubthread::waiter::wakeup() / wait_inf() / wait_for() / wait_until()"
tags:
  - component
  - cubrid
  - thread
  - daemon
  - concurrency
related:
  - "[[components/thread|thread]]"
  - "[[components/thread-manager|thread-manager]]"
  - "[[components/entry-task|entry-task]]"
  - "[[components/vacuum|vacuum]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/page-buffer|page-buffer]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cubthread::daemon` — Background Daemon Threads

`cubthread::daemon` is the abstraction for all long-running background threads in the CUBRID server. A daemon owns a `std::thread` and loops indefinitely, executing a task and sleeping between iterations according to a `looper` strategy.

> [!warning] SERVER_MODE only
> `cubthread::daemon` is guarded by `#if !defined(SERVER_MODE) #error`. Daemons are never created in SA_MODE; `manager::create_daemon` asserts false and returns NULL there.

## Daemon Internals

```cpp
class daemon {
  waiter   m_waiter;  // condition variable wrapper — wakeup signaling
  looper   m_looper;  // sleep strategy — determines next wait duration
  std::thread m_thread;
  std::string m_name;
  cubperf::statset& m_stats;
};
```

### Two Daemon Flavors

| Flavor | Constructor | Loop function | Entry consumption |
|--------|-------------|---------------|-------------------|
| With context | `daemon(looper, entry_manager*, entry_task*, name)` | `loop_with_context` | Claims 1 THREAD_ENTRY from pool |
| Without context | `daemon(looper, task<void>*, name)` | `loop_without_context` | No THREAD_ENTRY consumed |

**With-context loop** (`loop_with_context`):
1. Calls `entry_manager->create_context()` — claims a `THREAD_ENTRY`
2. Loops: `exec->execute(entry)` then `looper.put_to_sleep(waiter)`
3. On stop: `entry_manager->retire_context(entry)`

**Without-context loop** (`loop_without_context`):
1. Loops: `exec->execute()` then `looper.put_to_sleep(waiter)`
2. No `THREAD_ENTRY` involved — suitable for OS-level or non-CUBRID-state daemons

## `cubthread::looper` — Wait Strategies

The looper decides how long a daemon sleeps between task executions. Four patterns:

| `wait_type` | Constructor | Behavior |
|-------------|-------------|----------|
| `INF_WAITS` | `looper()` (default) | Sleep until explicit `wakeup()` — never times out |
| `FIXED_WAITS` | `looper(delta_time)` | Sleep exactly N ms/us each iteration |
| `INCREASING_WAITS` | `looper(array<delta_time, N>)` | Step through increasing periods on timeout; reset on wakeup |
| `CUSTOM_WAITS` | `looper(period_function)` | Caller-supplied function determines `(is_timed, period)` each iteration |

### Increasing-Periods Pattern

Useful for daemons that should be aggressive when there is work, but back off under idle conditions:

```cpp
// Wait 1ms, then 50ms, then 500ms on successive timeouts
std::array<delta_time, 3> periods = {{ 1ms, 50ms, 500ms }};
cubthread::looper loop(periods);
```

- If the daemon is woken up before timeout → period index **resets to 0** (become aggressive again)
- If timeout fires → period index advances to next level
- `daemon::reset_looper()` manually resets the index (call when significant work is found)

### `looper::put_to_sleep(waiter&)`

Internally: records `m_start_execution_time`, computes remaining sleep time as `period - task_execution_time`, then calls `waiter.wait_for(remaining)` or `waiter.wait_inf()`.

## `cubthread::waiter`

A thin wrapper over `std::condition_variable`:

```cpp
waiter::wakeup()       // notify_one() — wakes the daemon
waiter::wait_inf()     // wait until wakeup()
waiter::wait_for(d)    // wait for duration d or until wakeup — returns true if woken
waiter::wait_until(tp) // wait until time_point or until wakeup — returns true if woken
```

`daemon::wakeup()` calls `m_waiter.wakeup()` — typically called by the subsystem that has new work (e.g., log writer signals the log flush daemon).

## Daemon Lifecycle (via manager)

```cpp
// Create
REGISTER_DAEMON(my_daemon_name);   // at static init — accounts for 1 THREAD_ENTRY
daemon* d = cubthread::get_manager()->create_daemon(
    looper,          // wait strategy
    new MyTask(),    // entry_task* (ownership transferred)
    "my_daemon",     // name
    entry_mgr        // optional; defaults to manager's daemon_entry_manager
);

// Wakeup from another thread
d->wakeup();

// Destroy (stop loop, join thread, free entry)
cubthread::get_manager()->destroy_daemon(d);
```

`destroy_daemon` calls `daemon::stop_execution()` which sets the looper to stopped, wakes the daemon, and calls `m_thread.join()`.

## Known Daemons in CUBRID

| Daemon | Subsystem | Loop type | Wakeup trigger |
|--------|-----------|-----------|----------------|
| Vacuum master | [[components/vacuum\|vacuum]] | Increasing (fast when jobs available) | New MVCC GC work queued |
| Vacuum workers (pool) | [[components/vacuum\|vacuum]] | Worker pool tasks, not daemons | Dispatched by master |
| Log flush daemon | [[components/log-manager\|log-manager]] | Fixed or increasing | Log buffer threshold |
| Page flusher | [[components/page-buffer\|page-buffer]] | Increasing | LRU pressure / dirty threshold |
| Checkpoint daemon | [[components/log-manager\|log-manager]] | Fixed period | Timer |
| DWB flush | [[components/double-write-buffer\|double-write-buffer]] | Triggered | DWB buffer full |

> [!note] Vacuum uses a hybrid model
> The vacuum **master** is a daemon; it discovers work and dispatches tasks to a **worker pool**. Vacuum workers are `entry_task` instances in a pool, not individual daemons. This lets the worker count scale up via the pool while the master acts as the coordinator.

## Daemon Statistics

Each daemon tracks:
- Time spent executing (task execution duration)
- Time spent paused (sleep duration)
- Wakeup count

Accessible via `daemon::get_stats(stat_value*)`. Stat names via `daemon::get_stat_name(idx)`.

## Related

- [[components/thread|thread]] — parent namespace overview
- [[components/thread-manager|thread-manager]] — `create_daemon` / `destroy_daemon` factory
- [[components/entry-task|entry-task]] — the task type executed by daemons
- [[components/vacuum|vacuum]] — primary consumer of daemon infrastructure
- [[components/log-manager|log-manager]] — log flush and checkpoint daemons
- [[components/page-buffer|page-buffer]] — page flusher daemon
- Source: [[sources/cubrid-src-thread|cubrid-src-thread]]
