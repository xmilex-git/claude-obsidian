---
type: component
parent_module: "[[modules/src|src]]"
path: "src/loaddb/"
status: active
purpose: "Driver (orchestrates scanner + parser per batch), session (lifecycle + ordered commit), and worker manager (thread pool with driver pool)"
key_files:
  - "load_driver.cpp / load_driver.hpp"
  - "load_session.cpp / load_session.hpp"
  - "load_worker_manager.cpp / load_worker_manager.hpp"
  - "load_common.hpp (batch, batch_id, class_id types)"
tags:
  - component
  - cubrid
  - loaddb
  - driver
  - session
  - workers
related:
  - "[[components/loaddb|loaddb]]"
  - "[[components/loaddb-grammar|loaddb-grammar]]"
  - "[[components/loaddb-executor|loaddb-executor]]"
  - "[[components/thread|thread]]"
created: 2026-04-23
updated: 2026-04-23
---

# loaddb Driver, Session & Worker Manager

The control-plane of the loaddb subsystem. Three tightly coupled classes manage the lifecycle of a bulk load operation.

Hub: [[components/loaddb|loaddb]].

## `cubload::driver`

`driver` wires a `scanner` (flex) to a `parser` (bison) and holds references to the active `class_installer`, `object_loader`, and `error_handler` for the current batch.

```cpp
class driver {
  scanner             *m_scanner;
  class_installer     *m_class_installer;
  object_loader       *m_object_loader;
  error_handler       *m_error_handler;
  semantic_helper      m_semantic_helper;  // context state for grammar
  int                  m_start_line_no;
  bool                 m_is_initialized;
};
```

### Lifecycle

```
driver::initialize(cls_installer, obj_loader, error_handler)
  -- creates scanner; sets m_is_initialized = true

driver::parse(istream &iss, int line_offset)
  -- scanner->switch_streams(&iss)
  -- scanner->set_lineno(line_offset + 1)
  -- semantic_helper.reset_after_batch()
  -- parser parser(*this); parser.parse()   ← runs full bison parse
  -- returns 0 on success

driver::clear()
  -- deletes class_installer, object_loader, scanner, error_handler
  -- sets m_is_initialized = false
```

`driver` is **not** constructed fresh per batch; instead it is pooled in `resource_shared_pool<driver>` and `clear()` / `initialize()` is called between batches by the worker entry manager.

### `semantic_helper`

A helper object that tracks cross-line grammar state — e.g. whether the parser is currently inside an instance line (`set_in_instance_line`). Reset with `reset_after_line()` between rows and `reset_after_batch()` between batches.

## `cubload::session`

`session` is the root object for one `loaddb` run. Constructed on the server when the client connects.

```cpp
class session {
  std::mutex                 m_mutex;
  std::condition_variable    m_cond_var;
  std::vector<int>           m_tran_indexes;   // active worker transactions
  load_args                 &m_args;
  batch_id                   m_last_batch_id;  // last committed batch
  batch_id                   m_max_batch_id;   // highest batch seen
  std::atomic<int>           m_active_task_count;
  class_registry             m_class_registry;
  DB_CLIENT_TYPE             m_load_client_type;
  load_stats                 m_stats;
  std::atomic<bool>          m_is_failed;
  driver                    *m_driver;         // SA-mode driver
  // ...
};
```

### Ordered batch commit

This is the session's most important invariant: batches are **parsed in parallel** but **committed in order**.

```cpp
void session::wait_for_previous_batch(batch_id id)
{
  auto pred = [this, &id]() -> bool {
    return is_failed() || id == (m_last_batch_id + 1);
  };
  // blocks on m_cond_var until pred is true
}
```

After `xtran_server_commit`, the worker calls `notify_batch_done_and_register_tran_end(batch_id, tran_index)`, which increments `m_last_batch_id` and notifies all waiting workers.

This serialises commits without serialising parsing or heap insertion — all the slow work (parsing, string conversion, heap record building) runs in parallel.

### Session failure

`session::fail()` sets `m_is_failed = true`. All `load_task::execute()` calls check `m_session.is_failed()` at the start and return immediately. The session also calls `xtran_server_abort` on the current batch's transaction.

### Table-name CLI argument path

If `load_args.table_name` is non-empty (user passed `--table` on the CLI), the session constructor installs the class immediately before any batches arrive:

```cpp
m_driver->get_class_installer().set_class_id(FIRST_CLASS_ID);
m_driver->get_class_installer().install_class(args.table_name.c_str());
```

This allows the data file to omit `%class` directives entirely when loading a single table.

## `cubload::load_task`

`load_task` extends `cubthread::entry_task`. One is created per batch and submitted to the worker pool.

```cpp
void load_task::execute(cubthread::entry &thread_ref)
{
  // 1. Claim driver from thread_ref.m_loaddb_driver (set by worker_entry_manager::on_create)
  init_driver(driver, m_session);           // first use: initialize with installers
  // 2. Assign transaction
  logtb_assign_tran_index(&thread_ref, ...);
  // 3. Copy client IDs from session tdes to worker tdes
  worker_tdes->client.set_ids(session_tdes->client);
  // 4. Parse
  invoke_parser(driver, m_batch);           // driver->parse(istringstream(content), line_offset)
  // 5. Ordered commit or abort
  m_session.wait_for_previous_batch(m_batch.get_id());
  xtran_server_commit(&thread_ref, false);
  // 6. Update stats; notify done
  m_session.notify_batch_done_and_register_tran_end(batch_id, tran_index);
}
```

## `cubload::worker_entry_manager` and worker pool

```cpp
class worker_entry_manager : public cubthread::entry_manager {
  resource_shared_pool<driver> m_driver_pool;  // pool of pre-allocated drivers

  void on_create(cubthread::entry &ctx) {
    ctx.m_loaddb_driver = m_driver_pool.claim();  // claim one driver
    ctx.type = TT_LOADDB;
  }
  void on_retire(cubthread::entry &ctx) {
    ctx.m_loaddb_driver->clear();                 // reset driver
    m_driver_pool.retire(*ctx.m_loaddb_driver);   // return to pool
    ctx.m_loaddb_driver = NULL;
  }
};
```

Global state in `load_worker_manager.cpp`:

| Variable | Role |
|----------|------|
| `g_worker_pool` | `cubthread::stats_worker_pool_type*` — the actual thread pool |
| `g_wp_entry_manager` | `worker_entry_manager*` — manages per-thread driver assignment |
| `g_wp_task_capper` | `cubthread::worker_pool_task_capper*` — limits in-flight tasks |
| `g_active_sessions` | `std::set<session*>` — registered sessions |
| `g_wp_mutex` / `g_wp_condvar` | Global mutex/condvar for session registration |

`worker_manager_register_session(session&)` — called from `session` constructor. `worker_manager_unregister_session(session&)` — called from `session` destructor. `worker_manager_try_task(task*)` — submits a `load_task` to the pool.

The driver pool size equals the worker pool size; each worker always has exactly one `driver` available without contention.

## `cubload::batch` (load_common.hpp)

The unit of work passed from client to server. Implements `cubpacking::packable_object` for network serialisation.

```cpp
class batch : public cubpacking::packable_object {
  batch_id    m_id;           // sequence number (FIRST_BATCH_ID = 1)
  class_id    m_clsid;        // which class this batch belongs to
  std::string m_content;      // the raw text of the batch
  int64_t     m_line_offset;  // line number of first row in this batch (for error reporting)
  int64_t     m_rows;         // row count hint from client
};
```

`batch` is move-constructible but not copy-constructible — it is always transferred via `std::move` or managed by pointer.

## SA mode entry point

In `SA_MODE`, `load_sa_loader.cpp` drives the load directly without worker threads. It creates a `driver` with `sa_class_installer` and `sa_object_loader` (C-wrapper objects delegating to `load_object.c`), and calls `driver::parse()` directly for each block of input.

## Related

- [[components/loaddb|loaddb]] — hub and architecture overview
- [[components/loaddb-grammar|loaddb-grammar]] — what `driver::parse()` invokes
- [[components/loaddb-executor|loaddb-executor]] — the installer and loader implementations
- [[components/thread|thread]] — `cubthread::entry_task`, `entry_manager`, worker pool infrastructure
