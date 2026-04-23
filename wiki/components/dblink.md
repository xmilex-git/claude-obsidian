---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/"
status: active
purpose: "Remote CUBRID query execution via CCI (dblink_scan.c) and distributed 2-phase commit coordination (dblink_2pc.c)"
key_files:
  - "dblink_scan.c (remote query execution via CCI, S_DBLINK_SCAN backend)"
  - "dblink_scan.h (DBLINK_SCAN_INFO, DBLINK_CONN_ENTRY, DBLINK_HOST_VARS)"
  - "dblink_2pc.c (2-phase commit participant management)"
  - "dblink_2pc.h (2PC API: get_participants, send_prepare, end_tran)"
public_api:
  - "dblink_execute_query(thread_p, spec, vd, host_vars) → int"
  - "dblink_open_scan(thread_p, scan_info, spec, vd, host_vars) → int"
  - "dblink_scan_next(scan_info, val_list) → SCAN_CODE"
  - "dblink_scan_reset(scan_info) → SCAN_CODE"
  - "dblink_close_scan(scan_info) → int"
  - "dblink_end_tran(dblink, is_abort) → int"
  - "dblink_2pc_get_participants(thread_p, length, block_particps_ids) → int"
  - "dblink_2pc_send_prepare(thread_p, gtrid, num_partcps, block_particps_ids) → bool"
  - "dblink_2pc_end_tran(thread_p, gtrid, num_particps, is_commit, block_particps_ids)"
  - "dblink_2pc_dump_participants(fp, block_length, block_particps_ids)"
tags:
  - component
  - cubrid
  - dblink
  - distributed
  - 2pc
  - query
related:
  - "[[components/query|query]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# DBLink — Remote Query & 2-Phase Commit

Two tightly coupled files implement CUBRID's dblink feature: federated queries to remote CUBRID nodes (`dblink_scan.c`) and distributed transaction coordination (`dblink_2pc.c`).

## Remote Scan: `dblink_scan.c`

### Connection Model

DBLink uses the **CCI (CUBRID C Interface)** driver to connect to remote CUBRID nodes. Connections are represented as:

```c
struct dblink_conn_info {
  int  conn_handle;
  char conn_url[513];     // MAX_LEN_CONNECTION_URL + 1
  char user_name[...];
  char password[...];
};

struct dblink_conn_entry {
  DBLINK_CONN_INFO conn_info;
  bool is_2pc_participant;   // true if this conn is enrolled in 2PC
  DBLINK_CONN_ENTRY *next;   // linked list per transaction
};
```

`is_2pc_participant` marks which remote connections have been enlisted in the distributed transaction.

### Scan Info

```c
struct dblink_scan_info {
  int   conn_handle;    // CCI connection handle
  int   stmt_handle;    // CCI statement handle
  int   col_cnt;        // number of result columns
  char  cursor;         // cursor position (T_CCI_CURSOR_POS)
  void *col_info;       // T_CCI_COL_INFO array
};
```

### Scan Lifecycle

```
dblink_execute_query()     ← evaluate host vars, send SQL to remote
  ↓
dblink_open_scan()         ← bind scan_info to stmt handle
  ↓
dblink_scan_next() × N    ← fetch one row; map CCI values → val_list DB_VALUEs
  ↓
dblink_scan_reset()        ← rewind cursor (for nested loops)
  ↓
dblink_close_scan()        ← release CCI statement
```

### Host Variables

`DBLINK_HOST_VARS` carries the mapping from query host variable positions to the indices in the current `VAL_DESCR`:

```c
typedef struct { int count; int *index; } DBLINK_HOST_VARS;
```

The remote SQL is sent as-is; host variables are substituted by the CCI layer.

### Status Codes

| `DBLINK_STATUS` | Meaning |
|-----------------|---------|
| `DBLINK_SUCCESS` | Row fetched |
| `DBLINK_EOF` | Scan exhausted |
| `DBLINK_ERROR` | CCI error |

## Integration with Scan Manager

`scan_manager.c` includes `dblink_scan.h` and registers `S_DBLINK_SCAN` as a scan type. `scan_open_dblink_scan` wraps `dblink_open_scan`, and `scan_next_scan` dispatches to `dblink_scan_next`. The result feeds into the query executor exactly like a local heap scan row.

## 2-Phase Commit: `dblink_2pc.c`

When a transaction modifies data on a remote CUBRID node via dblink, it must coordinate commit/rollback using a two-phase commit protocol.

### Protocol

```
Phase 1 (PREPARE):
  dblink_2pc_get_participants()  ← collect enrolled DBLINK_CONN_ENTRY list
  dblink_2pc_send_prepare()      ← send PREPARE to each participant; returns false if any decline

Phase 2 (COMMIT or ROLLBACK):
  dblink_2pc_end_tran()          ← send COMMIT or ROLLBACK to all participants
```

`gtrid` is a global transaction ID assigned by the local transaction manager.

> [!key-insight] 2PC participants are linked-list per transaction
> Each `DBLINK_CONN_ENTRY` with `is_2pc_participant = true` is tracked in the transaction's dblink connection list. `dblink_2pc_get_participants` serializes these into a flat byte block (`block_particps_ids`) for the transaction manager's recovery log.

> [!warning] No distributed deadlock detection
> DBLink 2PC coordinates commit/rollback but does not implement cross-node wait-for graph detection. Deadlocks involving dblink connections require manual intervention or statement timeout.

### Dump

`dblink_2pc_dump_participants` prints the participant block to a `FILE *` for diagnostic use (e.g., from the transaction manager's recovery trace).

## Related

- Parent: [[components/query|query]]
- [[components/scan-manager|scan-manager]] — S_DBLINK_SCAN integration
- [[Query Processing Pipeline]] — dblink scan is a leaf node in the XASL tree
