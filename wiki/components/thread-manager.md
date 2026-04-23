---
type: component
parent_module: "[[modules/src|src]]"
path: "src/thread/"
status: stable
purpose: "Singleton thread manager: creates/destroys worker pools and daemons, owns the THREAD_ENTRY pool, bridges lock-free transaction system"
key_files:
  - "thread_manager.hpp (class definition, alias declarations, C shim functions)"
  - "thread_manager.cpp (implementation, global Manager/Main_entry_p statics)"
public_api:
  - "cubthread::get_manager() — returns the singleton cubthread::manager*"
  - "cubthread::initialize(entry*&) — creates Manager and Main_entry_p"
  - "cubthread::finalize() — tears down Manager and Main_entry_p"
  - "cubthread::get_entry() — returns current thread's entry (thread_local)"
  - "cubthread::is_single_thread() — true in SA_MODE"
  - "manager::create_worker_pool<Res>(pool_size, core_count, ...)"
  - "manager::destroy_worker_pool(Res*&)"
  - "manager::push_task(worker_pool*, entry_task*)"
  - "manager::push_task_on_core(worker_pool*, entry_task*, core_hash, method_mode)"
  - "manager::create_daemon(looper, entry_task*, name, entry_mgr=NULL)"
  - "manager::create_daemon_without_entry(looper, task<void>*, name)"
  - "manager::destroy_daemon(daemon*&)"
  - "manager::map_entries(func, args...) — iterate over all THREAD_ENTRY slots"
  - "manager::find_by_tid(thread_id_t) — lookup entry by OS thread id"
  - "manager::set_max_thread_count_from_config() — sum from REGISTER_* macros"
  - "thread_create_worker_pool(...) / thread_create_stats_worker_pool(...) — C shims"
tags:
  - component
  - cubrid
  - thread
  - concurrency
related:
  - "[[components/thread|thread]]"
  - "[[components/worker-pool|worker-pool]]"
  - "[[components/entry-task|entry-task]]"
  - "[[components/thread-daemon|thread-daemon]]"
  - "[[components/lockfree|lockfree]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cubthread::manager` — Thread Manager

`cubthread::manager` is the central registry for all threading resources in the CUBRID server. It is created as a process-global singleton by `cubthread::initialize()` and torn down by `cubthread::finalize()`.

## Singleton Lifecycle

```
cubthread::initialize(entry*& my_entry)
  os::resources::initialize()          // detect CPU topology
  new manager()                        // create singleton
  new entry() → Main_entry_p           // create main-thread entry (TT_MASTER, index=0)
  tl_Entry_p = Main_entry_p            // set thread-local pointer

cubthread::initialize_thread_entries()
  manager::set_max_thread_count_from_config()
  manager::alloc_entries()             // allocate m_all_entries[]
  manager::init_lockfree_system()      // lockfree::tran::system
  manager::init_entries(with_lock_free)

cubthread::finalize()
  delete Main_entry_p
  delete Manager
```

## Entry Pool

The manager pre-allocates a flat array of `cubthread::entry` objects at startup:

```cpp
m_all_entries = new entry[m_max_threads];
m_entry_dispatcher = new resource_shared_pool<entry>(m_all_entries, m_max_threads);
```

- `claim_entry()` — borrows one entry from the pool, saves to `tl_Entry_p`
- `retire_entry(entry&)` — returns entry to pool, clears `tl_Entry_p`
- Pool size cannot grow after init; if exhausted, new pools/daemons are refused (returns NULL)

## Pool/Daemon Registry

```cpp
// private members
std::vector<worker_pool*>  m_worker_pools;
std::vector<daemon*>       m_daemons;
std::vector<daemon*>       m_daemons_without_entries;
std::size_t                m_available_entries_count;
std::mutex                 m_entries_mutex;
```

`create_and_track_resource<Res>` is the generic template used for both pools and daemons:
1. Lock `m_entries_mutex`
2. Check `m_available_entries_count >= entries_count`
3. Subtract `entries_count` from available
4. `new Res(args...)` — construct resource
5. `tracker.push_back(new_res)`

`destroy_and_untrack_resource` reverses the process: calls `res->stop_execution()`, `delete res`, restores `entries_count` to available.

## Thread Count Configuration

```cpp
void manager::set_max_thread_count_from_config()
{
  m_max_threads = count_registry<connection>::total()
                + count_registry<worker_pool>::total()
                + count_registry<daemon>::total()
                + 1; // PAD
}
```

The three `count_registry` totals are populated at static initialization time by the `REGISTER_*` macros scattered across the codebase:

```cpp
REGISTER_WORKERPOOL(parallel_query, []() { return prm_get_integer_value(PRM_ID_MAX_PARALLEL_WORKERS); });
REGISTER_DAEMON(vacuum_master);
REGISTER_CONNECTION(client_connections, []() { return prm_get_integer_value(PRM_ID_MAX_CLIENTS); });
```

## `worker_pool_type` Alias

Declared in `thread_manager.hpp`:

```cpp
#if defined(SERVER_MODE)
  using worker_pool_type       = cubthread::worker_pool_impl<false>;  // no stats
  using stats_worker_pool_type = cubthread::worker_pool_impl<true>;   // with stats
#else
  using worker_pool_type       = cubthread::worker_pool;  // abstract (never used in SA)
  using stats_worker_pool_type = cubthread::worker_pool;
#endif
```

Callers use the `worker_pool_type` alias so they don't depend on the `Stats` template parameter directly.

## `push_task` Behavior

```cpp
void manager::push_task(worker_pool* pool, entry_task* task)
{
  if (pool == NULL) {
    task->execute(get_entry());   // inline — SA_MODE or NULL pool
    task->retire();
  } else {
    pool->execute(task);          // enqueue to worker pool
  }
}
```

`push_task_on_core` is the affinity variant: routes to a specific core by hash (used for method-mode execution where affinity matters).

## Lock-Free Transaction System

The manager owns and initializes the `lockfree::tran::system` used by all lock-free data structures:

```cpp
m_lf_tran_sys = new lockfree::tran::system(m_max_threads + 1);
```

Each `THREAD_ENTRY` gets an assigned LF transaction index via `m_lf_tran_sys->assign_index()` during `init_entries()`.

## C Compatibility Shims

`thread_manager.hpp` exposes inline C-callable wrappers:

| Shim | Maps to |
|------|---------|
| `thread_get_manager()` | `cubthread::get_manager()` |
| `thread_create_worker_pool(...)` | `get_manager()->create_worker_pool<worker_pool_type>(...)` |
| `thread_create_stats_worker_pool(...)` | `get_manager()->create_worker_pool<stats_worker_pool_type>(...)` |
| `thread_num_total_threads()` | `cubthread::get_max_thread_count()` |
| `thread_get_thread_entry_info()` | `&cubthread::get_entry()` |
| `thread_get_tran_entry(thread_p, idx)` | `thread_p->tran_entries[idx]` |
| `thread_sleep(double ms)` | `std::this_thread::sleep_for(...)` |

## Related

- [[components/thread|thread]] — parent hub page
- [[components/worker-pool|worker-pool]] — pool implementation details
- [[components/thread-daemon|thread-daemon]] — daemon creation and looper
- [[components/entry-task|entry-task]] — task base class and entry_manager
- [[components/lockfree|lockfree]] — lock-free transaction system managed here
- Source: [[sources/cubrid-src-thread|cubrid-src-thread]]
