---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/serial.{c,h}"
status: active
purpose: "SEQUENCE / SERIAL value generator: next_val cache, currval per-session, cyclic semantics"
key_files:
  - "serial.h (public xserial_* API)"
  - "serial.c (cache pool + catalog update path)"
public_api:
  - "xserial_get_current_value(thread_p, result_num, oid_p, cached_num)"
  - "xserial_get_next_value(thread_p, result_num, oid_p, cached_num, num_alloc, is_auto_increment, force_set_last_insert_id)"
  - "xserial_decache(thread_p, oidp) — invalidate entry on schema change"
  - "serial_initialize_cache_pool(thread_p) / serial_finalize_cache_pool() — boot / shutdown"
  - "serial_cache_index_btid(thread_p) / serial_get_index_btid(output) — B-tree index on _db_serial OID lookup"
tags:
  - component
  - cubrid
  - query
  - sequence
  - serial
related:
  - "[[modules/src|src]]"
  - "[[components/query]]"
  - "[[components/system-catalog]]"
  - "[[components/session]]"
  - "[[components/btree]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/query/serial.{c,h}` — SEQUENCE / SERIAL Generator

Implements CUBRID's `CREATE SERIAL` / `NEXT_VAL` / `CURRENT_VAL` feature — monotonically increasing (or decreasing) value generators backed by the `_db_serial` catalog table with a per-engine in-memory cache pool.

## Build-mode guard

```c
#if !defined (SERVER_MODE) && !defined (SA_MODE)
#error Belongs to server module
#endif
```

Server-side + standalone only. Client code reaches this through XASL + network protocol.

## Public entry points

| Function | Purpose |
|----------|---------|
| `xserial_get_current_value` | Returns `CURRVAL` — same value the calling **session** last fetched via `NEXTVAL`. Session-local semantics per [[components/session]]. |
| `xserial_get_next_value` | Returns `NEXTVAL` — bumps generator and returns the new value. Handles both manual `SELECT seq.NEXT_VAL` and auto-increment column path. |
| `xserial_decache` | Drops the cache entry for a given OID. Called on `DROP SERIAL` / `ALTER SERIAL` from [[components/schema-manager|schema-manager]]. |
| `serial_initialize_cache_pool` | Called from [[components/server-boot|server-boot]]. Sets up the per-engine shared cache and loads the `_db_serial` B-tree OID. |
| `serial_finalize_cache_pool` | Called at server shutdown. Releases all `SERIAL_CACHE_ENTRY`s. |

## Data structures

`SERIAL_CACHE_ENTRY` (in `serial.c`, not public):
- `inc_val`, `cur_val`, `min_val`, `max_val` — `DB_VALUE` (supports `NUMERIC` not just integers)
- `cyclic` flag
- refcount / activity tracking for eviction
- `cached_num` — remaining in the current batch

A **cache pool** = pool of pre-allocated `SERIAL_CACHE_ENTRY` slabs (`serial_alloc_cache_area`) to avoid per-request allocations.

## Execution path — `xserial_get_next_value` (fast path)

```
xserial_get_next_value
    │
    ├── look up cache entry by OID (hash-table hit?)
    │       ▲
    │       │  miss → fetch from _db_serial via btree_find + heap_fetch
    │       │          (serial_load_attribute_info_of_db_serial)
    │       │          → populate SERIAL_CACHE_ENTRY
    │
    ├── if cached_num > 0:
    │       cur_val += inc_val * num_alloc
    │       cached_num -= num_alloc
    │       return cur_val       ← NO DISK I/O, NO LOCK beyond latch
    │
    └── else (exhausted cache):
            serial_update_cur_val_of_serial()
                ├── LOCK _db_serial row (X)
                ├── read current cur_val from heap
                ├── advance by (cached_num_config × inc_val)
                ├── WAL: log_append_undoredo_data
                ├── heap_update_logical on catalog row
                └── refresh cache entry
```

## `serial_get_next_cached_value` — batch refill

When cache is exhausted, the next batch is computed based on:
- `cached_num` (configured per-SERIAL batch size, default from `CREATE SERIAL ... CACHED_NUM n`)
- `inc_val` × `cached_num` values are "reserved" in one catalog update
- subsequent NEXTVAL calls hand them out without I/O

> [!key-insight] Cache gap after crash
> Because a batch reservation commits the "post-batch" `cur_val` to `_db_serial` **before** handing out any of the reserved values, a crash mid-batch means all unhanded-out values are lost — **serial values are monotonic, not contiguous**. Applications that require gap-free sequences must set `CACHED_NUM 1`.

## Cyclic semantics — `serial_get_nth_value`

If the serial is declared `CYCLE`:
- When `cur_val + inc_val > max_val` (ascending) or `< min_val` (descending), wrap to `min_val` (asc) or `max_val` (desc).
- Non-cyclic serials raise `ER_QPROC_SERIAL_RANGE_OVERFLOW` on exhaustion.

## `CURRVAL` — session-local

`xserial_get_current_value` looks up the **session's** last-fetched value. Each session tracks its own NEXTVAL history via [[components/session-variables|session variables]] (`@@cur_val_of_X`). Calling `CURRVAL` before any `NEXTVAL` in the same session raises `ER_QPROC_SERIAL_NOT_FOUND`.

> [!key-insight] Session-local `CURRVAL`, cross-session `NEXTVAL`
> Two sessions can observe different `CURRVAL`s for the same serial at the same wall-clock moment (each is its own last-fetched value). `NEXTVAL` is globally monotonic across sessions but `CURRVAL` is session-private.

## Interaction with auto-increment columns

`xserial_get_next_value` takes `is_auto_increment` and `force_set_last_insert_id`:
- When INSERT triggers an auto-inc column, the value is generated here and also stashed in the session for `LAST_INSERT_ID()` (see [[components/session-variables]]).
- `force_set_last_insert_id = true` is used by bulk paths (e.g. [[components/loaddb]]) to explicitly set the per-session LAST_INSERT_ID even when multiple inserts occur in one call.

## Constraints

- **Threading**: cache pool uses latches, not full locks. Contention on hot serials is mitigated by the batch (`cached_num`) — a hot serial with `CACHED_NUM 1` will serialize all NEXTVAL calls at the catalog row.
- **Memory**: `SERIAL_CACHE_ENTRY` pre-allocated in slabs; entries are reused after decache.
- **Catalog integrity**: every batch refill is transactional (WAL'd). Uncommitted transactions that called NEXTVAL still consume values — they are NOT rolled back.
- **Error model**: returns `NO_ERROR` / `ER_FAILED` (+ `er_set`). See [[Error Handling Convention]].

## Lifecycle

```
Server boot  → serial_initialize_cache_pool → serial_cache_index_btid (load _db_serial BTID)
    │
    │  (during queries)
    ├── xserial_get_next_value × N → refill → XASL evaluation
    │
    │  (DDL)
    ├── DROP / ALTER SERIAL → xserial_decache(OID)
    │
    └── Server shutdown → serial_finalize_cache_pool
```

## Gotchas

> [!warning] NEXTVAL is never rolled back
> Even if the transaction that called `NEXTVAL` aborts, the value is already consumed. This is SQL-standard behavior but surprises users — aborted transactions leave gaps.

> [!warning] `serial_update_serial_object` latches the catalog page
> Concurrent refills for the same serial serialize on the `_db_serial` page latch. Under extreme contention this is the bottleneck.

## Related

- Parent: [[modules/src|src]]
- Hub: [[components/query]]
- Catalog: [[components/system-catalog]] (`_db_serial`)
- Per-session tracking: [[components/session]], [[components/session-variables]]
- Index lookup: [[components/btree]] (`serial_cache_index_btid`)
- Catalog mutation: [[components/heap-file|heap-file]] + WAL via [[components/log-manager|log-manager]]
