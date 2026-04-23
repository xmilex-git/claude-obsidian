---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/cursor.c"
status: active
purpose: "Client-side cursor over a QFILE_LIST_ID result spool; forwards/backwards tuple navigation and DB_VALUE extraction"
key_files:
  - "cursor.c (implementation)"
  - "cursor.h (CURSOR_ID struct and public API)"
public_api:
  - "cursor_open(cursor_id, list_id, updatable, oid_included) → bool"
  - "cursor_next_tuple(cursor_id) → int"
  - "cursor_prev_tuple(cursor_id) → int"
  - "cursor_first_tuple(cursor_id) → int"
  - "cursor_last_tuple(cursor_id) → int"
  - "cursor_get_tuple_value(cursor_id, index, value) → int"
  - "cursor_get_tuple_value_list(cursor_id, size, value_list) → int"
  - "cursor_get_current_oid(cursor_id, value) → int"
  - "cursor_close(cursor_id)"
  - "cursor_free(cursor_id)"
  - "cursor_set_prefetch_lock_mode(cursor_id, mode)"
  - "cursor_set_oid_columns(cursor_id, oid_col_no, cnt)"
  - "cursor_set_copy_tuple_value(cursor_id, copy) → bool"
tags:
  - component
  - cubrid
  - query
  - cursor
  - client
related:
  - "[[components/query|query]]"
  - "[[components/list-file|list-file]]"
  - "[[Query Processing Pipeline]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cursor.c` — Client-Side Cursor

The cursor is the client-facing iterator over a query result. It wraps a `QFILE_LIST_ID` (a server-produced temp list file) and exposes SQL-style positional navigation: BEFORE / ON / AFTER semantics with `cursor_next_tuple` / `cursor_prev_tuple` / `cursor_first_tuple` / `cursor_last_tuple`.

> [!key-insight] Client-side only
> `cursor.h` has **no** `SERVER_MODE`/`SA_MODE` guard — it is a client-side module. The server produces `QFILE_LIST_ID`; the client opens a cursor on it locally. In standalone (SA_MODE) both sides are in the same process, so the cursor reads pages from the temp volume directly.

## CURSOR_ID Structure

```c
struct cursor_id {
  QUERY_ID query_id;
  QFILE_LIST_ID list_id;        // the result list this cursor wraps
  OID  *oid_set;                // current page OID array (for updates)
  MOP  *mop_set;                // MOP cache for OID column values
  int   oid_ent_count;
  CURSOR_POSITION position;     // C_BEFORE | C_ON | C_AFTER
  VPID  current_vpid;           // current real page
  VPID  next_vpid;              // prefetch hint
  VPID  header_vpid;            // buffer area header page
  int   on_overflow;            // cursor buffer spans overflow page
  int   tuple_no;               // absolute tuple position
  QFILE_TUPLE_RECORD tuple_record;
  char *buffer;                 // current page buffer
  char *buffer_area;
  int   buffer_filled_size;
  int   buffer_tuple_count;
  int   current_tuple_no;       // position within page
  int   current_tuple_offset;
  char *current_tuple_p;
  int   current_tuple_length;
  int  *oid_col_no;             // column numbers holding OIDs
  int   oid_col_no_cnt;
  DB_FETCH_MODE prefetch_lock_mode;
  int   current_tuple_value_index;
  char *current_tuple_value_p;
  bool  is_updatable;
  bool  is_oid_included;        // first column is hidden OID
  bool  is_copy_tuple_value;    // copy (default) vs. peek
};
```

## Lifecycle

```
cursor_open()        ← binds CURSOR_ID to QFILE_LIST_ID
  └── position = C_BEFORE
cursor_next_tuple()  ← advance; returns DB_CURSOR_SUCCESS or DB_CURSOR_END
cursor_get_tuple_value(idx, db_value)
  └── reads column idx from current_tuple_p into DB_VALUE
cursor_close()       ← resets internal state (does NOT free list file)
cursor_free()        ← frees all owned memory (oid_set, mop_set, etc.)
```

## Bidirectional Navigation

| Function | Direction | Notes |
|----------|-----------|-------|
| `cursor_next_tuple` | forward | primary iterator |
| `cursor_prev_tuple` | backward | requires list file supports reverse |
| `cursor_first_tuple` | seek to start | |
| `cursor_last_tuple` | seek to end | |
| `cursor_fetch_page_having_tuple` | page seek | low-level; used internally |

## OID Columns and Updatable Cursors

`cursor_set_oid_columns` registers which output columns carry OID values. `is_oid_included` indicates the first column is a hidden OID added by the executor for `UPDATE WHERE CURRENT OF` support. `prefetch_lock_mode` controls the locking mode used when fetching OIDs from the MOP cache.

## Copy vs. Peek

`cursor_set_copy_tuple_value(cursor_id, false)` sets peek mode: `cursor_get_tuple_value` returns a pointer into the page buffer rather than copying into a fresh `DB_VALUE`. Faster but unsafe if the caller stores the value longer than the next cursor advance.

## List File Cleanup Macros

```c
#define cursor_free_list_id(list_id)       // frees last_pgptr, f_valp, sort_list, domp
#define cursor_free_self_list_id(list_id)  // above + free_and_init(list_id)
```

These are macros in `cursor.h` to avoid a function-call overhead for the hot path of result set cleanup.

## Related

- Parent: [[components/query|query]]
- [[components/list-file|list-file]] — the underlying `QFILE_LIST_ID` being iterated
- [[Query Processing Pipeline]] — cursor is the last stage: client reads the spool
- [[Build Modes (SERVER SA CS)]] — cursor is always client-side
