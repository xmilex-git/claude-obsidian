---
type: component
parent_module: "[[components/compat|compat]]"
path: "src/compat/db_admin.c, db_obj.c, db_query.c, db_vdb.c, db_set.c, db_date.c, db_json.cpp, db_elo.c"
status: active
purpose: "Client-side db_* API families: database lifecycle, connection, transaction, schema DDL, object CRUD, query compile/execute/fetch, set operations, LOB"
key_files:
  - "db_admin.c — database lifecycle: login, restart, shutdown, volume management, lock, isolation"
  - "db_obj.c — object API: create, get, put, drop, find_unique, send (method call)"
  - "db_query.c — query API: compile, execute, next_tuple, get_tuple_value, query_end"
  - "db_vdb.c — virtual DB / view API, deferred operations, prepare-execute protocol"
  - "db_set.c — set/multiset/sequence value operations"
  - "db_date.c — date, time, timestamp, datetime encode/decode helpers"
  - "db_json.cpp — JSON DB_VALUE support (deep copy, scalar conversion, schema validation)"
  - "db_elo.c — LOB (BLOB/CLOB) external large object API"
tags:
  - component
  - cubrid
  - client-api
  - compat
related:
  - "[[components/compat|compat]]"
  - "[[components/db-value|db-value]]"
  - "[[components/dbi-compat|dbi-compat]]"
  - "[[components/parser|parser]]"
  - "[[components/query-executor|query-executor]]"
  - "[[Error Handling Convention]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# Client API (`db_*` function families)

The `db_*` functions in `src/compat/` form CUBRID's public application-level API. They run entirely client-side (in `CS_MODE` client library or `SA_MODE` combined library). They are declared in `dbi_compat.h` and grouped by concern across several implementation files.

> [!key-insight] No server-side db_ functions
> In `SERVER_MODE`, the `db_admin.c` / `db_query.c` translation units are compiled with `#if !defined(SERVER_MODE)` guards around most significant code paths, or excluded entirely. The server never calls these functions; it evaluates XASL directly.

## Connection & Lifecycle (`db_admin.c`)

| Function | Description |
|----------|-------------|
| `db_login(name, password)` | Set credentials for next `db_restart` |
| `db_restart(program, print_version, volume)` | Open a named database; initialise all client subsystems |
| `db_restart_ex(program, db_name, db_user, db_password, preferred_hosts, client_type)` | Extended restart with explicit host list (for HA/broker use) |
| `db_shutdown()` | Flush dirty objects, commit pending work, disconnect |
| `db_ping_server(client_val, server_val)` | Liveness check |
| `db_disable_modification()` / `db_enable_modification()` | Toggle read-only mode for the session |
| `db_get_session_id()` / `db_set_session_id()` | Session state identity |

## Transaction (`db_admin.c`)

| Function | Description |
|----------|-------------|
| `db_commit_transaction()` | Commit current transaction |
| `db_abort_transaction()` | Roll back current transaction |
| `db_savepoint_transaction(name)` | Create named savepoint |
| `db_abort_to_savepoint(name)` | Partial rollback to savepoint |
| `db_commit_is_needed()` | True if dirty objects/open txn exist |
| `db_set_isolation(isolation)` | Set `DB_TRAN_ISOLATION` (SERIALIZABLE … READ_UNCOMMITTED) |
| `db_set_lock_timeout(seconds)` | Lock wait timeout (−1 = wait forever) |
| `db_2pc_start_transaction()` | 2PC phase 1 (distributed txn support) |
| `db_2pc_prepare_transaction()` | 2PC prepare-to-commit |
| `db_2pc_prepare_to_commit_transaction(gtrid)` | Commit after external coordinator agree |

## Schema DDL (`db_admin.c`, `db_obj.c`)

```
Class lifecycle:
  db_create_class(name)   →  DB_OBJECT*
  db_create_vclass(name)  →  DB_OBJECT* (view)
  db_drop_class(classobj)
  db_rename_class(classobj, new_name)

Attribute management:
  db_add_attribute(obj, name, domain, default_value)
  db_add_class_attribute(...)
  db_drop_attribute(classobj, name)
  db_change_default(classobj, name, value)
  db_constrain_non_null(classobj, name, class_attribute, on_or_off)
  db_constrain_unique(classobj, name, on_or_off)

Index / constraint:
  db_add_constraint(classmop, constraint_type, constraint_name, att_names, class_attributes)
  db_drop_constraint(...)
  db_add_index(classobj, attname)
  db_drop_index(classobj, attname)
```

## Object CRUD (`db_obj.c`)

```c
/* Create */
DB_OBJECT *db_create (DB_OBJECT *obj);            /* from class MOP */
DB_OBJECT *db_create_by_name (const char *name);  /* from class name */

/* Read */
int db_get (DB_OBJECT *object, const char *attpath, DB_VALUE *value);

/* Update */
int db_put (DB_OBJECT *obj, const char *name, DB_VALUE *value);

/* Delete */
int db_drop (DB_OBJECT *obj);

/* Fetch with lock mode */
int db_fetch_array (DB_OBJECT **objects, DB_FETCH_MODE mode, int quit_on_error);
int db_lock_read (DB_OBJECT *op);
int db_lock_write (DB_OBJECT *op);

/* Find by unique key */
DB_OBJECT *db_find_unique (DB_OBJECT *classobj, const char *attname, DB_VALUE *value);
DB_OBJECT *db_find_unique_write_mode (...);
DB_OBJECT *db_find_multi_unique (DB_OBJECT *classobj, int size,
                                  char *attnames[], DB_VALUE *values[],
                                  DB_FETCH_MODE purpose);
DB_OBJECT *db_find_primary_key (MOP classmop, const DB_VALUE **values,
                                 int size, DB_FETCH_MODE purpose);

/* Method call */
int db_send (DB_OBJECT *obj, const char *name, DB_VALUE *returnval, ...);
int db_send_arglist (DB_OBJECT *obj, const char *name,
                     DB_VALUE *returnval, DB_VALUE_LIST *args);
int db_send_argarray (DB_OBJECT *obj, const char *name,
                      DB_VALUE *returnval, DB_VALUE **args);
```

## Query Execution (`db_query.c`, `db_vdb.c`)

```
Compile → prepare statement:
  DB_SESSION *session = db_open_buffer(sql_text);
  int stmt_id = db_compile_statement(session);  /* → parses + type-checks + XASL gen */

Execute:
  DB_QUERY_RESULT *result;
  int error = db_execute_statement(session, stmt_id, &result);

Iterate results:
  while (db_query_next_tuple(result) == DB_CURSOR_SUCCESS)
    {
      DB_VALUE val;
      db_query_get_tuple_value(result, col_idx, &val);
      /* use val */
      db_value_clear(&val);
    }

Cleanup:
  db_query_end(result);
  db_close_session(session);
```

`DB_QUERY_RESULT` is an opaque client-side cursor into the result list returned by the server. `db_query_next_tuple` / `db_query_prev_tuple` / `db_query_seek_tuple` provide navigation. `db_query_get_tuple_value` fills a caller-provided `DB_VALUE`.

Column metadata available via `DB_QUERY_TYPE`:
```c
DB_QUERY_TYPE *col = db_get_query_type_list(result);
for (; col; col = db_query_type_next(col))
  {
    const char *name = db_query_column_name(col);
    DB_TYPE     type = db_query_column_type(col);
    int         size = db_query_column_size(col);
  }
```

## Error inspection

```c
int         db_error_code (void);            /* last error code (negative) */
const char *db_error_string (int level);     /* human-readable message */
int         db_error_init (const char *logfile); /* redirect error log */
```

These wrap `er_errid()` and `er_msg()` from the base error manager. The client never receives raw server error structs; error codes are translated through the network protocol and re-set on the client via `er_set`.

## Authorization (`db_admin.c`)

```c
DB_OBJECT  *db_get_user (void);
DB_OBJECT  *db_find_user (const char *name);
DB_OBJECT  *db_add_user (const char *name, int *exists);
int         db_drop_user (DB_OBJECT *user);
int         db_set_password (DB_OBJECT *user, const char *old, const char *new);
int         db_grant (DB_OBJECT *user, DB_OBJECT *obj, DB_AUTH auth, int grant_option);
int         db_revoke (DB_OBJECT *user, DB_OBJECT *obj, DB_AUTH auth);
int         db_check_authorization (DB_OBJECT *op, DB_AUTH auth);
```

`DB_AUTH` is a bitmask: `DB_AUTH_SELECT=1`, `DB_AUTH_INSERT=2`, `DB_AUTH_UPDATE=4`, `DB_AUTH_DELETE=8`, `DB_AUTH_ALTER=16`, `DB_AUTH_INDEX=32`, `DB_AUTH_EXECUTE=64`.

## Serial / sequence API

```c
int db_get_serial_current_value (const char *serial_name, DB_VALUE *serial_value);
int db_get_serial_next_value (const char *serial_name, DB_VALUE *serial_value);
int db_get_serial_next_value_ex (const char *serial_name, DB_VALUE *serial_value, int num_alloc);
```

## Set / collection operations (`db_set.c`, `db_set_function.h`)

```c
DB_SET *db_set_create (DB_OBJECT *classobj, const char *name);
DB_SET *db_set_create_basic (DB_OBJECT *classobj, const char *name);
DB_SET *db_set_create_multi (DB_OBJECT *classobj, const char *name);
DB_SEQ *db_seq_create (DB_OBJECT *classobj, const char *name, int size);

int db_set_add (DB_SET *set, DB_VALUE *value);
int db_set_get (DB_SET *set, int index, DB_VALUE *value);
int db_set_size (DB_SET *set);
int db_set_drop (DB_SET *set, DB_VALUE *value);
int db_set_free (DB_SET *set);     /* decrements refcount; frees if zero */

int db_seq_put (DB_SEQ *seq, int index, DB_VALUE *value);
int db_seq_get (DB_SEQ *seq, int index, DB_VALUE *value);
int db_seq_size (DB_SEQ *seq);
```

## LOB API (`db_elo.c`)

LOB (BLOB/CLOB) values are stored externally (`ELO_FBO` — file-based object). The `db_elo_*` functions manage the external storage lifecycle:

```c
int db_create_fbo (DB_VALUE *value, DB_TYPE type); /* allocate new LOB */
int db_elo_copy (DB_ELO *src, DB_ELO *dest);       /* deep copy */
int db_elo_delete (DB_ELO *elo);                   /* remove external file */
INT64 db_elo_size (DB_ELO *elo);                   /* size in bytes */
int db_elo_read (DB_ELO *elo, INT64 pos, void *buf, size_t count, INT64 *read_bytes);
int db_elo_write (DB_ELO *elo, INT64 pos, void *buf, size_t count, INT64 *written_bytes);
```

LOB values in `DB_VALUE` contain a `DB_ELO` struct (not inline data): a locator string + meta_data string + `DB_ELO_TYPE`. Full LOB contents are fetched on demand.

## SA_MODE vs. CS_MODE dispatch

`db_admin.c` includes `#if defined(SA_MODE)` guards for paths that call the storage/transaction layers directly. In CS_MODE, the same functions use `network_interface_cl.c` RPC calls. This is the only significant behavioral split within `compat/`.

## Related

- Parent: [[components/compat|compat]]
- [[components/db-value|db-value]] — `DB_VALUE` used throughout as the value type
- [[components/dbi-compat|dbi-compat]] — umbrella header declaring all these functions
- [[components/parser|parser]] — `db_compile_statement` calls into the parser pipeline
- [[components/query-executor|query-executor]] — server-side evaluation of compiled queries
- [[Error Handling Convention]] — `db_error_code` / `db_error_string` surface
- [[Build Modes (SERVER SA CS)]] — SA_MODE vs CS_MODE dispatch within these files
- Source: [[sources/cubrid-src-compat|cubrid-src-compat]]
