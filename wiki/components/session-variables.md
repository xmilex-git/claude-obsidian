---
type: component
parent_module: "[[modules/src|src]]"
path: "src/session/"
status: active
purpose: "@var user variable bindings and session-level system parameter overrides stored per connection"
key_files:
  - "session.c (SESSION_VARIABLE, session_add_variable, session_drop_variable, session_get_session_parameter)"
  - "session.h (session_set_session_variables, session_get_variable, session_define_variable, session_drop_session_variables, session_get_session_parameter, session_set_session_parameters)"
  - "src/base/system_parameter.c (SESSION_PARAM, prm_Def_session_idx, sysprm_get_session_parameters_count)"
tags:
  - component
  - cubrid
  - session
  - session-variables
  - system-parameter
  - server
related:
  - "[[components/session|session]]"
  - "[[components/session-state|session-state]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[components/sp-protocol|sp-protocol]]"
created: 2026-04-23
updated: 2026-04-23
---

# Session Variables & Parameter Overrides

Sub-component of [[components/session|session]]. This page covers both kinds of per-session mutable data that persist across transaction boundaries: user-defined `@var` bindings and system-parameter overrides.

## User-Defined Session Variables (`@var`)

### Storage

```c
struct session_variable {
  char *name;               // malloc'd string, case-insensitive key
  DB_VALUE *value;          // heap-allocated, deep-copied DB_VALUE
  SESSION_VARIABLE *next;   // singly linked list
};
```

`SESSION_STATE.session_variables` is the head of this list. Maximum 20 entries per session (`MAX_SESSION_VARIABLES_COUNT`); attempting to exceed this limit raises `ER_SES_TOO_MANY_VARIABLES`.

### Value Storage Rules

`db_value_alloc_and_copy()` handles deep copy of the value on set:

| Source DB_TYPE | Storage action |
|----------------|----------------|
| Numeric types (`INT`, `BIGINT`, `NUMERIC`, …) | `pr_clone_value()` |
| `CHAR`, `VARCHAR`, `BIT`, `VARBIT` | `malloc` + `memcpy` of the string buffer |
| Any other type | Coerce to `VARCHAR` via `tp_value_cast()`, then store as varchar |

On update (`update_session_variable()`), the old string buffer is `free_and_init()`'d before the new value is copied in.

### API

| Function | SQL construct |
|----------|---------------|
| `session_set_session_variables(thread_p, values, count)` | `SET @a = 1, @b = 2` |
| `session_define_variable(thread_p, name, value, result)` | `SET @x := expr` (returns value) |
| `session_get_variable(thread_p, name, result)` | `SELECT @x` (copies value to result) |
| `session_get_variable_no_copy(thread_p, name, &result)` | SA_MODE only — returns raw pointer |
| `session_drop_session_variables(thread_p, values, count)` | internal (no SQL syntax for drop) |

`session_get_variable_no_copy()` is explicitly asserted false in SERVER_MODE — it returns a direct pointer into the session state which is unsafe in a multi-threaded context.

### Magic Variable Side Effects

`session_add_variable()` intercepts two special names before storing:

| Name (case-insensitive) | Side effect |
|------------------------|-------------|
| `collect_exec_stats` | value=1 → `perfmon_start_watch(NULL)`; value=0 → `perfmon_stop_watch(NULL)` |
| `trace_plan` | Stores the value string in `state_p->plan_string` (enables plan-trace output via `session_get_trace_stats()`) |

Both side effects trigger regardless of whether the variable is successfully added to the list.

## System Parameter Overrides (`SESSION_PARAM`)

### Concept

CUBRID system parameters (defined in [[components/system-parameter|system-parameter]]) can be overridden at session scope. The session stores only the **delta** — parameters that differ from the global defaults. The client sends a batch of `SESSION_PARAM` values at connect/re-connect time via `session_set_session_parameters()`.

### Storage

```c
SESSION_PARAM *session_parameters;  // flat array, owned by SESSION_STATE
```

The array is indexed via `prm_Def_session_idx[PARAM_ID]` — a compile-time table that maps each `PARAM_ID` to its slot in the session parameter array, or `-1` if that parameter is not session-overridable.

### Lookup (O(1))

```c
SESSION_PARAM *session_get_session_parameter(THREAD_ENTRY *thread_p, PARAM_ID id)
{
  int idx = prm_Def_session_idx[id];
  return (idx < 0) ? NULL : &session_p->session_parameters[idx];
}
```

Callers check for NULL to know whether the session has an override for a given parameter.

### Lifecycle

- **Set**: `session_set_session_parameters(thread_p, session_parameters)` stores the array pointer directly (caller transfers ownership).
- **Free**: `sysprm_free_session_parameters(&session_parameters)` in `session_state_uninit()`.
- **PL bridge**: `session_set_pl_session_parameter(thread_p, id)` (SERVER_MODE only) propagates a parameter from the Java PL engine back into the session delta. Called by the SP protocol layer when the PL engine changes a session-level system parameter. See [[components/sp-protocol|sp-protocol]].

### Which Parameters Are Session-Overridable?

Determined by `sysprm_get_session_parameters_count()` and `prm_Def_session_idx[]` in `src/base/system_parameter.c`. Examples typically include: query plan cache enable/disable, timezone, isolation level defaults, and debug/trace flags. The set is defined in the system_parameter module, not in session code.

## Scoping Summary

```
Global system parameter (cubrid.conf)
  └── Session override (SESSION_PARAM array in SESSION_STATE)
        └── @var user variable (SESSION_VARIABLE linked list in SESSION_STATE)
              (no further narrowing — @vars are session-level, not transaction-level)
```

Neither `@var` values nor session parameter overrides are rolled back on `ROLLBACK`. They survive transaction boundaries.

## Related

- Hub: [[components/session|session]]
- [[components/session-state|session-state]] — the enclosing struct
- [[components/system-parameter|system-parameter]] — global param definitions, `SESSION_PARAM` type, `prm_Def_session_idx`
- [[components/sp-protocol|sp-protocol]] — `session_set_pl_session_parameter()` bridge
