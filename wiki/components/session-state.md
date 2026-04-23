---
type: component
parent_module: "[[modules/src|src]]"
path: "src/session/"
status: active
purpose: "SESSION_STATE struct definition, ACTIVE_SESSIONS hashmap, create/check/destroy lifecycle, timeout reaper daemon"
key_files:
  - "session.c (struct SESSION_STATE, session_state_create, session_state_destroy, session_check_session)"
  - "session.h (public API declarations)"
tags:
  - component
  - cubrid
  - session
  - server
related:
  - "[[components/session|session]]"
  - "[[components/session-variables|session-variables]]"
  - "[[components/connection|connection]]"
  - "[[components/transaction|transaction]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/lockfree|lockfree]]"
created: 2026-04-23
updated: 2026-04-23
---

# Session State Struct & Lifecycle

Sub-component of [[components/session|session]]. This page covers the internal `SESSION_STATE` struct, the global `ACTIVE_SESSIONS` container, and the full create/check/destroy lifecycle.

## Struct Layout

`SESSION_STATE` is defined only inside `session.c` (not in the public header):

```c
struct session_state {
  SESSION_ID id;                 // uint32 monotone key
  SESSION_STATE *stack;          // lockfree freelist link
  SESSION_STATE *next;           // hashmap chain
  pthread_mutex_t mutex;         // per-entry lock
  UINT64 del_id;                 // epoch for lock-free reclaim

  bool is_keep_session;          // survive disconnect (pool reconnect)
  bool is_trigger_involved;      // freeze insert-ID inside triggers
  bool is_last_insert_id_generated;
  bool auto_commit;

  DB_VALUE cur_insert_id;        // in-flight INSERT serial value
  DB_VALUE last_insert_id;       // last committed INSERT serial value
  int row_count;                 // rows affected by last DML (-1 = unset)

  SESSION_VARIABLE *session_variables;    // @var linked list (max 20)
  PREPARED_STATEMENT *statements;         // prepared stmt list (max 20)
  SESSION_QUERY_ENTRY *queries;           // holdable cursor results

  time_t active_time;            // last heartbeat (timeout reaper)
  SESSION_PARAM *session_parameters;      // array of param overrides
  char *trace_stats;             // query trace JSON/text
  char *plan_string;             // cached query plan (SET @trace_plan)
  int trace_format;              // QUERY_TRACE_TEXT | QUERY_TRACE_JSON

  int ref_count;                 // # CSS connections using this state
  TZ_REGION session_tz_region;   // per-session timezone
  int private_lru_index;         // page-buffer LRU partition (-1=none)

  load_session *load_session_p;  // bulk load session (if active)
  PL_SESSION *pl_session_p;      // Java PL engine session context
};
```

## Global Container

```c
typedef struct active_sessions {
  session_hashmap_type states_hashmap;   // lockfree_hashmap<SESSION_ID, session_state>
  SESSION_ID last_session_id;            // ATOMIC_INC_32 counter
  int num_holdable_cursors;              // global count across all sessions
} ACTIVE_SESSIONS;

static ACTIVE_SESSIONS sessions;         // process-global singleton
```

Hash size: 1000 buckets. Hash function: `id % hash_table_size`. Session IDs are monotone-increasing uint32 values; collision on wrap-around is handled by `session_key_increment()` which just increments until a free slot is found.

## Lifecycle State Machine

```
           session_states_init()
                   │
          [ACTIVE_SESSIONS ready]
                   │
     ┌─────────────▼──────────────┐
     │  session_state_create()    │
     │  – assign ID               │
     │  – insert into hashmap     │
     │  – new PL_SESSION(id)      │
     │  – pgbuf_assign_private_lru│
     │  – cache in conn_entry     │
     └──────────┬─────────────────┘
                │
         [STATE: ACTIVE]
                │
     ┌──────────▼─────────────────┐
     │ session_check_session()    │◄── every request start
     │  – refresh active_time     │
     │  – swap conn_entry pointer │
     │  – update ref_count        │
     └──────────┬─────────────────┘
                │
      ┌─────────▼─────────────────┐
      │ session_state_destroy()   │◄── disconnect or timeout reaper
      │  is_keep_session=true?    │
      │    └── mark + return      │
      │  ref_count > 0?           │
      │    └── defer teardown     │
      │  else:                    │
      │    session_state_uninit() │
      │    hashmap.erase_locked() │
      └───────────────────────────┘
```

## ref_count Protocol (SERVER_MODE)

- `session_state_increase_ref_count()` — called at `session_check_session()` and `session_state_create()`.
- `session_state_decrease_ref_count()` — called when a request finishes or an old session is displaced by a new `session_check_session()` call on the same thread.
- If `ref_count > 0` when `session_state_destroy()` is called, the state is kept alive until the last holder calls `session_state_decrease_ref_count()`.

## Timeout Reaper

Daemon: `session_control_daemon` — 60-second fixed interval (`cubthread::looper`).

Algorithm in `session_remove_expired_sessions()`:
1. Iterate all sessions via `session_hashmap_iterator`.
2. For each session, call `session_check_timeout()`:
   - Elapsed since `active_time` ≥ `PRM_ID_SESSION_STATE_TIMEOUT` (integer seconds)?
   - If yes (SERVER_MODE): fetch `css_get_session_ids_for_active_connections()` (lazily, once per sweep), check if session still has an active CSS connection — if so, reset `active_time` and keep.
3. Buffer expired IDs (max 1024 per pass).
4. After finishing the iteration pass, call `states_hashmap.erase()` for each buffered ID.
5. Loop until no new expired sessions found.

The two-pass approach is required because `lf_hash_delete` resets the lock-free transaction, which would invalidate the iterator.

## Related

- Hub: [[components/session|session]]
- [[components/connection|connection]] — `CSS_CONN_ENTRY.session_p` caches the session pointer
- [[components/lockfree|lockfree]] — `lockfree_hashmap` used for the session hash
- [[components/page-buffer|page-buffer]] — `pgbuf_assign_private_lru` / `pgbuf_release_private_lru`
- [[components/transaction|transaction]] — `logtb_set_current_user_active()` called at attach/detach
