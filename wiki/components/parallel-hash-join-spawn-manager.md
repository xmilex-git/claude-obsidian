---
status: developing
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_hash_join/px_hash_join_spawn_manager.hpp"
path_impl: "src/query/parallel/px_hash_join/px_hash_join_spawn_manager.cpp"
tags:
  - component
  - cubrid
  - parallel
  - query
  - hash-join
related:
  - "[[components/parallel-hash-join|parallel-hash-join]]"
  - "[[components/parallel-hash-join-task-manager|parallel-hash-join-task-manager]]"
  - "[[components/xasl|xasl]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_hash_join_spawn_manager` — Worker-Thread XASL Structure Spawner

`spawn_manager` is a per-worker-thread singleton (TLS) responsible for lazily spawning copies of XASL execution structures needed by `join_task`. It wraps `cubxasl::spawner` and caches the spawned structures so each worker re-uses them across multiple contexts processed in a single task execution.

## Purpose

`join_task::execute()` processes multiple `HASHJOIN_CONTEXT` objects in a loop. Each context needs its own `VAL_DESCR`, `PRED_EXPR`, and `REGU_VARIABLE_LIST` pointers so worker threads do not corrupt the originals. `spawn_manager` lazily creates one copy of each per worker thread and reuses it for all contexts handled by that thread.

## Class / Function Inventory

| Symbol | Kind | Description |
|--------|------|-------------|
| `spawn_manager` | class | TLS singleton, owns the spawner and all spawned structs |
| `spawn_manager(thread_ref)` | ctor | Initialises all fields to nullptr |
| `~spawn_manager()` | dtor | Explicitly calls `m_spawner->~spawner()` then `db_private_free_and_init` |
| `get_instance(thread_ref)` | static | Lazily creates the TLS instance via `db_private_alloc` + placement new; returns nullptr on OOM |
| `destroy_instance()` | static | Calls dtor + `db_private_free_and_init` on the TLS instance; called at end of `join_task::execute()` |
| `get_thread_ref()` | accessor | Returns `m_thread_ref` |
| `get_val_descr(src)` | spawner | Returns spawned `VAL_DESCR*`; must be called first |
| `get_during_join_pred(src)` | spawner | Returns spawned `PRED_EXPR*` |
| `get_outer_regu_list_pred(src)` | spawner | Returns spawned outer `REGU_VARIABLE_LIST` |
| `get_inner_regu_list_pred(src)` | spawner | Returns spawned inner `REGU_VARIABLE_LIST` |
| `get_spawner()` | private | Lazily creates `cubxasl::spawner`; uses `db_private_alloc` + placement new |
| `spawn<T>(src, dest)` | private template | Idempotent: if `dest != nullptr` returns existing; otherwise calls `spawner->spawn(src)` |

### Key Fields

| Field | Type | Role |
|-------|------|------|
| `m_thread_ref` | `cubthread::entry&` | Owning thread (for `db_private_alloc`) |
| `m_spawner` | `cubxasl::spawner*` | Underlying XASL structure deep-copier |
| `m_val_descr` | `VAL_DESCR*` | Cached spawned value descriptor |
| `m_during_join_pred` | `PRED_EXPR*` | Cached spawned join predicate |
| `m_outer_regu_list_pred` | `REGU_VARIABLE_LIST` | Cached outer predicate regu list |
| `m_inner_regu_list_pred` | `REGU_VARIABLE_LIST` | Cached inner predicate regu list |
| `tls_spawn_manager` | `inline static thread_local` | The TLS pointer; `nullptr` until `get_instance` |

## Execution Path

```
join_task::execute(thread_ref)
  │
  ├─ spawn_manager::get_instance(thread_ref)   ← creates TLS instance if needed
  │    └─ db_private_alloc(sizeof(spawn_manager))
  │       placement_new<spawn_manager>(raw, thread_ref)
  │
  ├─ [loop: get_next_context()]
  │    ├─ context->val_descr = spawn_manager->get_val_descr(manager->val_descr)
  │    │    └─ spawn<VAL_DESCR>(src, m_val_descr)
  │    │         └─ first call: get_spawner() → spawner->spawn(src)
  │    │            subsequent calls: return m_val_descr directly (cached)
  │    ├─ context->during_join_pred = get_during_join_pred(...)
  │    ├─ context->outer.regu_list_pred = get_outer_regu_list_pred(...)
  │    ├─ context->inner.regu_list_pred = get_inner_regu_list_pred(...)
  │    ├─ hjoin_execute(thread_ref, manager, context)
  │    └─ context->val_descr = nullptr   ← zero out, not freed (owned by spawn_manager)
  │
  └─ spawn_manager::destroy_instance()
       └─ ~spawn_manager()
            └─ m_spawner->~spawner(); db_private_free_and_init(&m_thread_ref, m_spawner)
```

> [!key-insight] `get_val_descr` must be called before other spawn methods
> The comment in the header states: _"get_val_descr must be called first, because it creates a DB_VALUE reused by other spawned structures."_ The spawner's internal DB_VALUE pool is initialised when `VAL_DESCR` is spawned; the predicate and regu-list spawners then reference the same pool.

> [!key-insight] One spawn per worker, N re-uses across contexts
> The `spawn<T>` template is idempotent: if `dest != nullptr` it immediately returns the existing pointer. This means the first context incurs the copy cost; all subsequent contexts in the same worker task get zero-cost pointer returns.

## Constraints

- **Thread ownership**: `spawn_manager` is a TLS singleton. It must only be accessed from the owning worker thread. `get_instance` asserts (implicitly via pointer null-check) that only one instance exists per thread.
- **Memory ownership**: all fields (`m_spawner`, `m_val_descr`, etc.) are allocated with `db_private_alloc` on `m_thread_ref`'s private heap. They are freed by the destructor.
- **Exception safety**: `get_instance` and `get_spawner` catch `...` exceptions from placement new, free the raw memory, set the error code via `assert_release_error`, and return nullptr.
- **No copy/move**: all four special members (copy ctor, copy assign, move ctor, move assign) are deleted.
- **Build modes**: requires `db_private_alloc` — active in `SERVER_MODE` and `SA_MODE`.

## Lifecycle

```
1. join_task::execute() begins on worker thread
2. get_instance(thread_ref) — lazy singleton creation
3. first get_val_descr() call — spawner is created, VAL_DESCR cloned
4. subsequent get_* calls — predicate + regu-lists cloned (once, cached)
5. per-context loop: pointers assigned to context, hjoin_execute called, pointers zeroed
6. task ends: destroy_instance() — full teardown, db_private_free
7. tls_spawn_manager reset to nullptr
```

## Related

- [[components/parallel-hash-join|parallel-hash-join]] — parent hub
- [[components/parallel-hash-join-task-manager|parallel-hash-join-task-manager]] — `join_task` calls this class
- [[components/xasl|xasl]] — `cubxasl::spawner` is defined in `xasl_spawner.hpp`
- [[Memory Management Conventions]] — `db_private_alloc` + explicit free pattern
