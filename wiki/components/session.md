---
type: component
parent_module: "[[modules/src|src]]"
path: "src/session/"
status: active
purpose: "Per-connection session state container: session variables, prepared statements, holdable cursors, session parameters, PL session, query trace"
key_files:
  - "session.h — public API"
  - "session.c — SESSION_STATE struct, ACTIVE_SESSIONS hash, lifecycle + all feature implementations"
public_api:
  - "session_states_init(thread_p)"
  - "session_states_finalize(thread_p)"
  - "session_state_create(thread_p, *id)"
  - "session_state_destroy(thread_p, id, is_keep_session)"
  - "session_check_session(thread_p, id)"
  - "session_set_session_variables(thread_p, values, count)"
  - "session_get_variable(thread_p, name, result)"
  - "session_define_variable(thread_p, name, value, result)"
  - "session_drop_session_variables(thread_p, values, count)"
  - "session_create_prepared_statement(thread_p, name, alias_print, sha1, info, info_len)"
  - "session_get_prepared_statement(thread_p, name, info, info_len, xasl_entry)"
  - "session_delete_prepared_statement(thread_p, name)"
  - "session_set_session_parameters(thread_p, session_parameters)"
  - "session_get_session_parameter(thread_p, id)"
  - "session_store_query_entry_info(thread_p, qentry_p)"
  - "session_load_query_entry_info(thread_p, qentry_p)"
  - "session_remove_query_entry_info(thread_p, query_id)"
  - "session_get_last_insert_id(thread_p, value, update_last_insert_id)"
  - "session_set_cur_insert_id(thread_p, value, force)"
  - "session_get_row_count(thread_p, row_count)"
  - "session_set_row_count(thread_p, row_count)"
  - "session_get_trace_stats(thread_p, result)"
  - "session_set_trace_stats(thread_p, stats, format)"
  - "session_get_session_tz_region(thread_p)"
  - "session_get_pl_session(thread_p, pl_session_ref_ptr)"
tags:
  - component
  - cubrid
  - session
  - server
related:
  - "[[modules/src|src]]"
  - "[[components/transaction|transaction]]"
  - "[[components/connection|connection]]"
  - "[[components/cas|cas]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[components/cursor|cursor]]"
  - "[[components/sp|sp]]"
  - "[[components/sp-jni-bridge|sp-jni-bridge]]"
  - "[[components/session-state|session-state]]"
  - "[[components/session-variables|session-variables]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/session/` — Per-Connection Session State

The session layer is a single-file subsystem (`session.c` / `session.h`) that acts as the **per-connection state container** for the CUBRID server. Every connection managed by a CAS worker gets exactly one `SESSION_STATE` object. That object survives across transactions for the lifetime of the logical connection, holding state that is broader than any individual transaction but narrower than the server process.

> [!key-insight] Session vs transaction lifetime
> A session lives from client connect to client disconnect (or timeout). A transaction lives only from `BEGIN` / auto-commit to `COMMIT` / `ROLLBACK`. The session carries data that must persist across transaction boundaries: `@var` user variables, prepared statement metadata, holdable cursor result lists, session-level parameter overrides, and the current/last insert ID.

## Architecture Overview

```
  CAS worker process
        │
        ▼
  CSS_CONN_ENTRY (connection layer)
     .session_id  ──────────────────────► ACTIVE_SESSIONS.states_hashmap
     .session_p   ──────────────────────► SESSION_STATE (cached pointer)
        │
        ▼
  THREAD_ENTRY.private_lru_index ◄──── SESSION_STATE.private_lru_index
                                         (own page-buffer LRU partition)

  ACTIVE_SESSIONS (global singleton)
  ┌────────────────────────────────────────────────────────────┐
  │ states_hashmap  cubthread::lockfree_hashmap<SESSION_ID,    │
  │                  session_state>  (1000-bucket hash)        │
  │ last_session_id  monotone counter (ATOMIC_INC_32)          │
  │ num_holdable_cursors  global holdable cursor count         │
  └────────────────────────────────────────────────────────────┘
```

## SESSION_STATE Struct

Defined internally in `session.c` (not exposed in the header):

| Field | Type | Role |
|-------|------|------|
| `id` | `SESSION_ID` | Unique numeric ID (monotone uint32) |
| `mutex` | `pthread_mutex_t` | Per-state lock |
| `ref_count` | `int` | Active CSS connections referencing this state |
| `is_keep_session` | `bool` | Survive disconnect (reconnect without re-login) |
| `is_trigger_involved` | `bool` | Inhibits insert-ID promotion inside trigger context |
| `auto_commit` | `bool` | Per-session auto-commit override |
| `cur_insert_id` | `DB_VALUE` | Insert ID of the current (in-progress) INSERT |
| `last_insert_id` | `DB_VALUE` | Insert ID of the last completed INSERT |
| `row_count` | `int` | Rows affected by the most recent DML (`-1` = unset) |
| `session_variables` | `SESSION_VARIABLE *` | Linked list of `@name = value` bindings |
| `statements` | `PREPARED_STATEMENT *` | Linked list of prepared statement metadata |
| `queries` | `SESSION_QUERY_ENTRY *` | Linked list of holdable cursor query results |
| `session_parameters` | `SESSION_PARAM *` | Array of per-session system parameter overrides |
| `trace_stats` | `char *` | JSON or text query trace output |
| `plan_string` | `char *` | Cached query plan text (`SET @trace_plan`) |
| `trace_format` | `int` | `QUERY_TRACE_TEXT` or `QUERY_TRACE_JSON` |
| `session_tz_region` | `TZ_REGION` | Per-session timezone |
| `private_lru_index` | `int` | Page buffer private LRU partition (-1 = none) |
| `active_time` | `time_t` | Last heartbeat for timeout eviction |
| `load_session_p` | `load_session *` | Reference to bulk-load session (when loaddb active) |
| `pl_session_p` | `PL_SESSION *` | Reference to Java PL engine session state |

## Sub-Components

| Area | Wiki page |
|------|-----------|
| Struct & lifecycle | [[components/session-state]] |
| `@var` user variables & param overrides | [[components/session-variables]] |

## Lifecycle

### Init / Finalize (server-wide)

```
boot_sr.c:
  session_states_init(thread_p)
    └── sessions.states_hashmap.init(...)  // 1000-bucket lock-free hashmap
    └── session_control_daemon_init()      // 60-second reaper daemon

  session_states_finalize(thread_p)
    └── session_control_daemon_destroy()
    └── sessions.states_hashmap.destroy()
```

### Create / Check / Destroy (per connection)

```
CAS connect:
  session_state_create(thread_p, &id)
    ├── ATOMIC_INC_32(&sessions.last_session_id)  // assign ID
    ├── states_hashmap.insert(thread_p, *id, session_p)
    ├── new PL_SESSION(id)                         // PL engine state
    ├── pgbuf_assign_private_lru(thread_p)         // own LRU partition
    └── session_set_conn_entry_data(thread_p, ...)  // cache in conn_entry

CAS request start:
  session_check_session(thread_p, id)
    ├── Drop old cached session_p from conn_entry
    ├── Lookup new id in hashmap
    ├── session_p->active_time = time(NULL)         // reset timeout
    └── Re-cache in conn_entry (increase ref_count)

CAS disconnect:
  session_state_destroy(thread_p, id, is_keep_session)
    ├── If is_keep_session → mark flag, return       // pool reconnect
    ├── Decrease ref_count; if ref_count > 0 → defer
    └── session_state_uninit(session_p)              // free all sub-fields
          ├── session_stop_attached_threads()
          ├── delete pl_session_p
          ├── free session_variables linked list
          ├── free statements linked list
          ├── free queries (holdable cursors)
          ├── sysprm_free_session_parameters()
          └── pgbuf_release_private_lru()
```

### Timeout Reaper

A `session-control` daemon thread (`cubthread::daemon`, 60-second `looper`) runs `session_remove_expired_sessions()`. It:
1. Iterates all sessions in the hash.
2. Calls `session_check_timeout()` — compares `active_time` against `PRM_ID_SESSION_STATE_TIMEOUT`.
3. In SERVER_MODE, queries `css_get_session_ids_for_active_connections()` to allow grace for still-connected sessions.
4. Calls `session_state_uninit()` then `states_hashmap.erase()` on confirmed-expired sessions.

> [!warning] Two-pass delete required
> The reaper cannot call `lf_hash_delete` while iterating (resetting the lock-free transaction would corrupt the iterator). It buffers up to 1024 expired IDs in `expired_sid_buffer`, finishes the iteration pass, then erases them in a second pass.

## Prepared Statement Cache

Each session holds a **singly-linked list** of `PREPARED_STATEMENT` nodes (cap: `MAX_PREPARED_STATEMENTS_COUNT = 20`):

```c
struct prepared_statement {
  char *name;         // user-supplied PREPARE name
  char *alias_print;  // canonical printed form (used as XASL cache key)
  SHA1Hash sha1;      // SHA-1 of alias_print  →  xcache lookup key
  int info_length;
  char *info;         // serialized XASL plan bytes (client-side column info)
  PREPARED_STATEMENT *next;
};
```

`session_get_prepared_statement()` does two lookups:
1. Finds `PREPARED_STATEMENT` by name (case-insensitive, linked list scan).
2. Calls `xcache_find_sha1(thread_p, &stmt_p->sha1, ...)` to retrieve the global XASL cache entry.

> [!key-insight] Prepared statements are session-local; the compiled plan (XASL) is server-global
> The `PREPARED_STATEMENT` node lives in the session and stores metadata + raw info bytes. The actual XASL tree lives in the server-global XASL cache (`xasl_cache.c`) and is fetched by SHA-1 hash. If the XASL entry was evicted, the PREPARE must be re-executed — the session metadata is stale but not an error, since `alias_print == NULL` or the xcache returns NULL triggers a fallback path.

## Holdable Cursor Store

After `CLOSE CURSOR`, if the query is declared `WITH HOLD`, the query manager entry is transferred to a `SESSION_QUERY_ENTRY`:

```
session_store_query_entry_info(thread_p, qentry_p)
  ├── Create SESSION_QUERY_ENTRY from QMGR_QUERY_ENTRY
  ├── session_preserve_temporary_files()  // file_temp_preserve per VFID
  └── Prepend to state_p->queries list; sessions.num_holdable_cursors++
```

The holdable result survives transaction commit and is restored by `session_load_query_entry_info()` when the client re-opens the cursor. Closing the stored entry calls `qfile_close_list()` + `qmgr_free_temp_file_list()`.

## Session Parameters (System Param Overrides)

`SESSION_PARAM` is defined in `system_parameter.h`. The session stores an **array** of overridden system parameters (only the subset that has been changed by the client). `session_get_session_parameter(thread_p, id)` uses `prm_Def_session_idx[id]` for O(1) lookup into the array.

The PL/SP bridge calls `session_set_pl_session_parameter(thread_p, id)` (SERVER_MODE only) when the Java PL engine needs to propagate a parameter change back to the session delta. See [[components/sp-protocol]].

## Session Variables (`@var`)

User-defined variables (`SET @x = 1; SELECT @x`) are stored as a linked list of `SESSION_VARIABLE` nodes (cap: `MAX_SESSION_VARIABLES_COUNT = 20`):

```c
struct session_variable {
  char *name;          // malloc'd, case-insensitive comparison
  DB_VALUE *value;     // heap-allocated DB_VALUE with deep copy
  SESSION_VARIABLE *next;
};
```

- Values are deep-copied via `db_value_alloc_and_copy()`: strings are `malloc`'d, numerics are `pr_clone_value()`'d.
- Non-string / non-numeric types are coerced to `VARCHAR` on storage.
- `MAX_SESSION_VARIABLES_COUNT = 20` (hard cap; `ER_SES_TOO_MANY_VARIABLES` if exceeded).
- Two magic variable names trigger side effects in `session_add_variable()`:
  - `collect_exec_stats` → calls `perfmon_start_watch()` / `perfmon_stop_watch()`
  - `trace_plan` → stores value in `state_p->plan_string` (enables plan tracing)

For the full variable reference see [[components/session-variables]].

## Session Accessor Pattern (SERVER_MODE)

In SERVER_MODE, `session_get_session_state(thread_p)` is a **zero-hash-lookup hot path** — it just dereferences `thread_p->conn_entry->session_p` (a direct pointer cached at `session_check_session()` time). The hashmap is consulted only at session create/check/destroy. This avoids lock contention on every per-session state read.

In SA_MODE, the function falls back to `sessions.states_hashmap.find(thread_p, id)` on each call.

## Private LRU Partition

Each session is assigned a dedicated page-buffer LRU partition via `pgbuf_assign_private_lru()`. The index is stored in both `SESSION_STATE.private_lru_index` and `THREAD_ENTRY.private_lru_index`. This gives each connection its own hot cache slice, avoiding cross-connection eviction pressure. Released on session destroy via `pgbuf_release_private_lru()`.

## PL Engine Integration

`SESSION_STATE.pl_session_p` holds a `PL_SESSION *` (declared in `pl_session.hpp`). It is created as `new PL_SESSION(id)` during `session_state_create()` and `delete`'d in `session_state_uninit()`. It encapsulates Java PL engine per-connection state (execution context, open SP connections, etc.). See [[components/sp]] and [[components/sp-jni-bridge]].

## Build Mode

`session.h` is SERVER_MODE + SA_MODE only (`#error Belongs to server module` guard). In SA_MODE, mutex operations are all no-ops via `#define pthread_mutex_*`. `session_get_variable_no_copy()` asserts false in SERVER_MODE (SA_MODE only — unsafe in multi-threaded context).

## Common Modification Points

| Task | Function | File |
|------|----------|------|
| Add session-level state | Add field to `SESSION_STATE`; initialize in `session_state_init()`; free in `session_state_uninit()` | `session.c` |
| Change prepared statement cap | `MAX_PREPARED_STATEMENTS_COUNT` | `session.c` |
| Change session variable cap | `MAX_SESSION_VARIABLES_COUNT` | `session.c` |
| Change timeout behavior | `session_check_timeout()` | `session.c` |
| Add param to session-level override | `system_parameter.c` (`prm_Def_session_idx`) | `src/base/` |
| Add magic @var side-effect | `session_add_variable()` | `session.c` |

## Gotchas

- `is_keep_session = true` defers session teardown so connection pools can hand the session to the next client without re-authenticating. The session timeout checker skips `is_keep_session` sessions.
- `is_trigger_involved = true` freezes both `cur_insert_id` and `last_insert_id` — inside a trigger, `LAST_INSERT_ID()` returns the outer statement's value, not the trigger's.
- `session_get_variable_no_copy()` returns a raw `DB_VALUE *` pointer into the session state — safe only in SA_MODE (single-threaded).
- The XASL cache (`xcache_find_sha1`) can evict entries independently of the session. A stale `PREPARED_STATEMENT` with a valid `sha1` but no XASL cache hit is a normal condition, not a bug.
- Holdable cursor cleanup during session destroy calls `qfile_close_list()` which requires the file manager to be alive — session finalize must happen before file/storage finalize.

## Related

- Parent: [[modules/src|src]]
- [[components/transaction|transaction]] — sessions hold state across multiple transactions; `logtb_set_current_user_active()` called at session attach/detach
- [[components/connection|connection]] — `CSS_CONN_ENTRY` caches `session_p`; `css_get_session_ids_for_active_connections()` used by timeout reaper
- [[components/cas|cas]] — CAS calls `session_state_create()` on connect and `session_state_destroy()` on disconnect
- [[components/system-parameter|system-parameter]] — `SESSION_PARAM`, `prm_Def_session_idx`, `sysprm_free_session_parameters()`
- [[components/cursor|cursor]] — holdable cursors stored as `SESSION_QUERY_ENTRY` in session
- [[components/sp|sp]] — `PL_SESSION` embedded in session state
- [[components/sp-jni-bridge|sp-jni-bridge]] — `session_set_pl_session_parameter()` propagates param deltas
- [[components/page-buffer|page-buffer]] — private LRU assigned per session
- [[components/session-state|session-state]] — struct & lifecycle detail
- [[components/session-variables|session-variables]] — @var binding & param override detail
- Source: [[sources/cubrid-src-session]]
