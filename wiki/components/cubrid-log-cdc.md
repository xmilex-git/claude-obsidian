---
type: component
parent_module: "[[components/api|api]]"
path: "src/api/cubrid_log.h, src/api/cubrid_log.c"
status: active
purpose: "Change Data Capture (CDC) client API: subscribe to CUBRID WAL changes, extract DDL/DML/DCL events as typed log items, replay with LSA-based positioning"
key_files:
  - "cubrid_log.h — public header: all types and function declarations"
  - "cubrid_log.c — implementation (CS_MODE only): four-phase state machine, CSS-level transport"
tags:
  - component
  - cubrid
  - cdc
  - change-data-capture
  - log
  - api
related:
  - "[[components/api|api]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/transaction|transaction]]"
  - "[[components/connection|connection]]"
  - "[[components/compat|compat]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Error Handling Convention]]"
created: 2026-04-23
updated: 2026-04-23
---

# CDC Client API (`cubrid_log.h / cubrid_log.c`)

The **CUBRID Log CDC API** gives external C programs a typed, LSA-positioned stream of every committed DML, DDL, and transaction lifecycle event recorded in the WAL. It lives entirely in `src/api/` and is the primary mechanism for building replication pipelines, audit trails, and ETL connectors against a live CUBRID server.

> [!key-insight] Dedicated raw connection, not broker
> `cubrid_log_connect_server` opens a raw `CSS_CONN_ENTRY` directly to `cub_server` (bypassing the broker/CAS layer) and speaks the `NET_SERVER_CDC_*` sub-protocol. This gives it access to log supplemental information that the standard SQL path does not expose.

## Four-Phase State Machine

The entire API is stateful. The internal `CUBRID_LOG_STAGE` enum enforces call ordering:

```
CONFIGURATION → PREPARATION → EXTRACTION → (FINALIZATION → CONFIGURATION)
```

| Phase | Allowed calls |
|-------|--------------|
| `CONFIGURATION` | `cubrid_log_set_*` configuration setters, then `cubrid_log_connect_server` |
| `PREPARATION` | `cubrid_log_find_lsa`, `cubrid_log_extract` (first call) |
| `EXTRACTION` | `cubrid_log_extract` (repeated), `cubrid_log_clear_log_item` |
| `FINALIZATION` | `cubrid_log_finalize` (from PREPARATION or EXTRACTION) |

Calling a function in the wrong phase returns `CUBRID_LOG_INVALID_FUNC_CALL_STAGE (-27)`.

## Public API

### Configuration Step

```c
int cubrid_log_set_connection_timeout (int timeout);   /* 0–360 sec; default 300 */
int cubrid_log_set_extraction_timeout (int timeout);   /* 0–360 sec; default 300 */
int cubrid_log_set_tracelog (char *path, int level, int filesize); /* level 0|1; size 8–512 MB */
int cubrid_log_set_max_log_item (int max_log_item);    /* 1–1024; default 512 */
int cubrid_log_set_all_in_cond (int retrieve_all);     /* 0=filter, 1=all */
int cubrid_log_set_extraction_table (uint64_t *classoid_arr, int arr_size);
int cubrid_log_set_extraction_user  (char **user_arr,  int arr_size);
```

`cubrid_log_set_extraction_table` and `cubrid_log_set_extraction_user` install server-side filters: the CDC session will only stream changes for the specified class OIDs and/or usernames. If both are omitted (or `retrieve_all=1`), all changes are delivered.

### Preparation Step

```c
int cubrid_log_connect_server (char *host, int port, char *dbname,
                                char *id, char *password);
int cubrid_log_find_lsa (time_t *timestamp, uint64_t *lsa);
```

`cubrid_log_connect_server` does three things in sequence:
1. Calls `db_restart` + `au_login` to verify the caller is a **DBA-group member**, then `db_shutdown`.
2. Opens a raw CSS connection to `host:port` (`css_connect_to_log_server`).
3. Sends `NET_SERVER_CDC_START_SESSION` with all configured parameters; fails with `CUBRID_LOG_UNAVAILABLE_CDC_SERVER` if the server's `supplemental_log` parameter is off.

`cubrid_log_find_lsa` sends `NET_SERVER_CDC_FIND_LSA` with a Unix timestamp and receives the nearest valid `LOG_LSA`. Returns `CUBRID_LOG_SUCCESS_WITH_ADJUSTED_LSA (2)` when the exact timestamp had no log record and the server adjusted forward.

### Extraction Step

```c
int cubrid_log_extract (uint64_t *lsa, CUBRID_LOG_ITEM **log_item_list, int *list_size);
int cubrid_log_clear_log_item (CUBRID_LOG_ITEM *log_item_list);
```

`cubrid_log_extract` is the hot loop:
1. Sends `NET_SERVER_CDC_GET_LOGINFO_METADATA` with the current LSA → receives next LSA + item count + total byte length.
2. If items exist, sends `NET_SERVER_CDC_GET_LOGINFO` → receives the raw packed blob.
3. Unpacks the blob into an array of `CUBRID_LOG_ITEM` (reusing `g_log_items` / `g_log_infos` globals; grown with `realloc` as needed).
4. Updates `*lsa` in-place to the new next-LSA for the caller to persist and re-supply on the next call.

`cubrid_log_clear_log_item` frees per-DML `malloc`'d arrays (`changed_column_index`, `changed_column_data`, etc.) but leaves the backing `g_log_items` and `g_log_infos` buffers alive for reuse.

### Finalization Step

```c
int cubrid_log_finalize (void);
```

Sends `NET_SERVER_CDC_END_SESSION`, frees the CSS connection, and resets all globals back to defaults. Stage returns to `CONFIGURATION`, allowing re-use without process restart.

## Data Types

### `CUBRID_LOG_ITEM` — linked list node

```c
struct cubrid_log_item {
  int              transaction_id;
  char            *user;           /* zero-copy pointer into g_log_infos blob */
  int              data_item_type; /* DATA_ITEM_TYPE_DDL/DML/DCL/TIMER */
  CUBRID_DATA_ITEM data_item;      /* tagged union */
  CUBRID_LOG_ITEM *next;
};
```

### `CUBRID_DATA_ITEM` — tagged union

| Variant | Type | Key fields |
|---------|------|-----------|
| `DDL` | schema change | `ddl_type`, `object_type`, `oid`, `classoid`, `statement` |
| `DML` | row change | `dml_type`, `classoid`, `num_changed_column`, column indexes + data arrays, `num_cond_column` + condition arrays |
| `DCL` | commit / rollback | `dcl_type`, `timestamp` |
| `TIMER` | heartbeat tick | `timestamp` |

### `DML` column encoding

DML column data is deserialized from a packed binary format using a `pack_func_code` discriminant (0=int, 1=int64, 2=float, 3=double, 4=short, 5=string, 7=nullable string, 8=string). Column data pointers (`changed_column_data[i]`, `cond_column_data[i]`) are **zero-copy pointers into the `g_log_infos` blob** except for the index/length arrays which are individually `malloc`'d and freed in `cubrid_log_clear_log_item`.

## Error Codes

The CDC API uses its own negative error namespace (`CUBRID_LOG_*`) separate from CUBRID's `ER_*` engine codes:

| Code | Value | Meaning |
|------|-------|---------|
| `CUBRID_LOG_SUCCESS` | 0 | OK |
| `CUBRID_LOG_SUCCESS_WITH_NO_LOGITEM` | 1 | No items at requested LSA (wait and retry) |
| `CUBRID_LOG_SUCCESS_WITH_ADJUSTED_LSA` | 2 | LSA adjusted to nearest valid position |
| `CUBRID_LOG_UNAVAILABLE_CDC_SERVER` | −34 | `supplemental_log` param off on server |
| `CUBRID_LOG_FAILED_LOGIN` | −33 | Auth failure or not DBA |
| `CUBRID_LOG_EXTRACTION_TIMEOUT` | −6 | Server-side extraction timed out |
| `CUBRID_LOG_INVALID_FUNC_CALL_STAGE` | −27 | API called in wrong phase |
| `CUBRID_LOG_FAILED_MALLOC` | −28 | Memory allocation failure |
| `CUBRID_LOG_LSA_NOT_FOUND` | −7 | Timestamp has no corresponding log record |

## Threading Model

The implementation is **not thread-safe**. All state is in file-scope globals (`g_stage`, `g_conn_entry`, `g_next_lsa`, `g_log_infos`, `g_log_items`, etc.). The API is designed for a **single-threaded CDC consumer loop**. If multi-session CDC is needed, separate processes are required.

## `supplemental_log` Dependency

The server must be configured with `supplemental_log = 1` in `cubrid.conf` before the CDC session will succeed. Without it, `cubrid_log_send_configurations` receives `ER_CDC_NOT_AVAILABLE` and returns `CUBRID_LOG_UNAVAILABLE_CDC_SERVER`. This parameter gates whether the transaction layer writes the supplemental log records (column-level before/after images) that CDC reads from the WAL.

## Typical Usage Pattern

```c
/* 1. Configure */
cubrid_log_set_connection_timeout(60);
cubrid_log_set_extraction_timeout(30);
cubrid_log_set_max_log_item(256);

/* 2. Connect */
cubrid_log_connect_server("localhost", 30000, "mydb", "dba", "");

/* 3. Find starting LSA */
time_t ts = time(NULL) - 3600;   /* 1 hour ago */
uint64_t lsa;
cubrid_log_find_lsa(&ts, &lsa);

/* 4. Extract loop */
CUBRID_LOG_ITEM *items;
int count;
while (running) {
    int rc = cubrid_log_extract(&lsa, &items, &count);
    if (rc == CUBRID_LOG_SUCCESS) {
        for (CUBRID_LOG_ITEM *it = items; it; it = it->next) {
            /* process it->data_item_type / it->data_item */
        }
        cubrid_log_clear_log_item(items);
        persist_lsa(lsa);  /* checkpoint for restart */
    } else if (rc == CUBRID_LOG_SUCCESS_WITH_NO_LOGITEM) {
        sleep(1);          /* poll — no items yet */
    } else {
        handle_error(rc);
    }
}

/* 5. Finalize */
cubrid_log_finalize();
```

## Trace Logging

`cubrid_log_set_tracelog(path, level, filesize_mb)` enables a rotating two-file trace log (named `<dbname>_cubridlog_<timestamp>.err`). Level 1 emits function entry/exit with input/output values. The rotation keeps at most two files; older files are removed automatically.

## Related

- Parent: [[components/api|api]]
- [[components/log-manager|log-manager]] — server WAL that CDC extracts from
- [[components/transaction|transaction]] — `supplemental_log` parameter and log record types
- [[components/connection|connection]] — `CSS_CONN_ENTRY` and `css_connect_to_log_server`
- [[components/compat|compat]] — `db_restart` / `au_login` used during the DBA check in connect
- [[Build Modes (SERVER SA CS)]] — entire file guarded `#if defined(CS_MODE)`
- Source: [[sources/cubrid-src-api|cubrid-src-api]]
