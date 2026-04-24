---
type: component
parent_module: "[[components/object|object]]"
path: "src/object/"
status: active
purpose: "Transaction-aware LOB locator state machine: tracks BLOB/CLOB locator strings through create/bind/delete lifecycle"
key_files:
  - "lob_locator.cpp (state machine implementation)"
  - "lob_locator.hpp (interface: lob_locator_state enum + 6 functions)"
public_api:
  - "lob_locator_find(locator, real_locator) → LOB_LOCATOR_STATE"
  - "lob_locator_add(locator, state) → int"
  - "lob_locator_change_state(locator, new_locator, state) → int"
  - "lob_locator_drop(locator) → int"
  - "lob_locator_is_valid(locator) → bool"
  - "lob_locator_key(locator) → const char* (pointer into locator string)"
  - "lob_locator_meta(locator) → const char* (pointer into locator string)"
tags:
  - component
  - cubrid
  - lob
  - blob
  - clob
  - client
related:
  - "[[components/object|object]]"
  - "[[components/external-storage|external-storage]]"
  - "[[components/storage|storage]]"
  - "[[components/transaction|transaction]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# LOB Locator (`src/object/lob_locator.cpp`)

The LOB locator module tracks the per-transaction lifecycle of BLOB and CLOB locator strings. A locator is an opaque string that names a specific external file managed by [[components/external-storage|src/storage/es.c]].

> [!info] LOB handling is a cross-cutting concern
> The locator state machine lives here (client side, `src/object/`). The physical byte storage is in [[components/external-storage|src/storage/es.c]] (POSIX or OWFS backend). Heap operations (`heap_attrinfo_delete_lob` in `heap_file.c`) bridge the two layers by detecting LOB attributes and calling `es_delete_file`. Any behavioral change touches multiple files.

## Locator String Structure

A locator string has the form `<meta><PATH_SEPARATOR><key>.<extension>`:
- `lob_locator_key(locator)` — returns pointer to the last `.`+1 (the key suffix).
- `lob_locator_meta(locator)` — returns pointer to the last `PATH_SEPARATOR`.
- `lob_locator_is_valid(locator)` — validates that both pointers exist and are in the right order.

## State Machine

A locator transitions through `LOB_LOCATOR_STATE`:

```
LOB_UNKNOWN            — unknown / out-of-transaction state

Within a transaction:

  LOB_TRANSIENT_CREATED  — locator created in this txn, not yet bound to a row
  LOB_PERMANENT_CREATED  — locator created AND bound to a table row in this txn
  LOB_PERMANENT_DELETED  — locator was bound and has now been deleted in this txn
  LOB_TRANSIENT_DELETED  — locator existed before this txn, deleted in this txn

LOB_NOT_FOUND          — lookup miss (not in the locator registry)
```

Key state transitions:

| Scenario | Transition |
|----------|------------|
| s1: Create then delete (same txn, never bound) | `LOB_TRANSIENT_CREATED → LOB_UNKNOWN` |
| s2: Create then bind to row | `LOB_TRANSIENT_CREATED → LOB_PERMANENT_CREATED` |
| s3: Bound then deleted (same txn) | `LOB_PERMANENT_CREATED → LOB_PERMANENT_DELETED` |
| s4: Delete a pre-existing locator | `LOB_UNKNOWN → LOB_TRANSIENT_DELETED` |

> [!key-insight] Rollback semantics
> On transaction rollback:
> - `LOB_TRANSIENT_CREATED` / `LOB_PERMANENT_CREATED` → the external file must be deleted (it was created in this txn and is now orphaned).
> - `LOB_TRANSIENT_DELETED` → the external file must be restored (delete is undone).
> - `LOB_PERMANENT_DELETED` → the external file is kept (the row insert was also rolled back).

## Implementation: Mode Dispatch

`lob_locator.cpp` dispatches based on build mode:

```cpp
// CS_MODE — client side of client/server split
LOB_LOCATOR_STATE lob_locator_find (const char *locator, char *real_locator) {
#if defined(CS_MODE)
  return log_find_lob_locator (locator, real_locator);   // network call to server
#else
  return xtx_find_lob_locator (NULL, locator, real_locator); // SA_MODE direct call
#endif
}
```

The actual registry lives in `transaction_transient.hpp` on the server side (or in-process for SA_MODE). The locator module here is a thin client-side wrapper.

## API Summary

| Function | Purpose |
|----------|---------|
| `lob_locator_find(locator, real_locator)` | Look up state; if `LOB_PERMANENT_CREATED`, `real_locator` is filled with final path |
| `lob_locator_add(locator, state)` | Register a new locator with initial state |
| `lob_locator_change_state(locator, new_locator, state)` | Transition state (and optionally rename locator) |
| `lob_locator_drop(locator)` | Remove from registry |
| `lob_locator_is_valid(locator)` | Validate locator string format |
| `lob_locator_key(locator)` | Extract key portion of locator string |
| `lob_locator_meta(locator)` | Extract meta portion of locator string |

## Common Bug Locations

| Symptom | Cause |
|---------|-------|
| LOB file orphaned after rollback | `LOB_TRANSIENT_CREATED` not cleaned up on abort path |
| LOB file deleted on commit | State machine stuck at `LOB_TRANSIENT_DELETED` instead of advancing to `LOB_UNKNOWN` |
| Locator string parse failure | `lob_locator_meta` or `lob_locator_key` returns `NULL` when `PATH_SEPARATOR` or `.` is missing from locator |
| LOB missing after SELECT | `lob_locator_find` returning `LOB_NOT_FOUND` when locator is in `real_locator` form (needs `change_state` call first) |

## Related

- Parent: [[components/object|object]]
- [[components/external-storage|external-storage]] — physical byte storage: `es_create_file`, `es_write_file`, `es_delete_file`
- [[components/storage|storage]] — `heap_attrinfo_delete_lob` bridges heap and LOB locator
- [[components/transaction|transaction]] — locator registry lives in `transaction_transient`; rollback triggers locator cleanup
- [[Build Modes (SERVER SA CS)]] — CS_MODE: network call; SA_MODE: direct `xtx_*` call
