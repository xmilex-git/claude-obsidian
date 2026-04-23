---
type: component
parent_module: "[[components/transaction|transaction]]"
path: "src/transaction/boot_sr.c, boot_sr.h, boot_cl.c"
status: active
purpose: "Server boot sequence: subsystem initialization order, database creation, crash recovery entry point"
key_files:
  - "boot_sr.c (server boot: boot_restart_server, boot_create_all_volumes, subsystem init)"
  - "boot_sr.h (BOOT_SERVER_STATUS, boot_Server_status, BOOT_DB_PARM)"
  - "boot_cl.c (client-side boot/connection for CS_MODE)"
public_api:
  - "boot_restart_server(thread_p, print_restart, db_name, from_backup, r_args) → int"
  - "boot_shutdown_server(thread_p, is_er_final) → bool"
  - "xboot_initialize_server(...) — first-time database creation"
  - "xboot_restart_server(...) — server restart (called by cub_server main)"
tags:
  - component
  - cubrid
  - boot
  - startup
  - transaction
  - server
related:
  - "[[components/transaction|transaction]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/recovery|recovery]]"
  - "[[components/storage|storage]]"
  - "[[components/vacuum|vacuum]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# Server Boot

`src/transaction/boot_sr.c` owns the server-side startup and shutdown sequence. It initializes every subsystem in strict dependency order and serves as the single entry point for both fresh database creation and crash recovery on restart.

## `BOOT_SERVER_STATUS`

Global variable `boot_Server_status` tracks the server lifecycle:

```c
typedef enum boot_server_status BOOT_SERVER_STATUS;
enum boot_server_status {
  BOOT_SERVER_DOWN,
  BOOT_SERVER_UP,
  BOOT_SERVER_MAINTENANCE
};
```

Any subsystem can query `boot_Server_status` to guard operations that must not run during startup or shutdown.

## `BOOT_DB_PARM` — Persistent Database Parameters

Stored in a dedicated heap record at the root class page:

```c
struct boot_dbparm {
  VFID  trk_vfid;              /* file tracker */
  HFID  hfid;                  /* this record's heap (validation) */
  HFID  rootclass_hfid;        /* heap of all class definitions */
  EHID  classname_table;       /* extendible hash: class name → OID */
  CTID  ctid;                  /* system catalog file */
  VFID  query_vfid;            /* query temp file (TODO: remove) */
  char  rootclass_name[10];
  OID   rootclass_oid;
  VOLID nvols;                  /* number of data volumes */
  VOLID temp_nvols;             /* number of temp volumes */
  VOLID last_volid;
  VOLID temp_last_volid;
  int   vacuum_log_block_npages;
  VFID  vacuum_data_vfid;       /* vacuum tracking data */
  VFID  dropped_files_vfid;     /* vacuum dropped files list */
  HFID  tde_keyinfo_hfid;       /* TDE encryption key info */
};
```

This struct survives across restarts — it is the root of the database's file and schema topology.

## Boot Sequence (Restart)

```
cub_server main()
  └── xboot_restart_server(...)
        └── boot_restart_server(thread_p, print_restart, db_name, from_backup, r_args)
```

**Strict initialization order** (abbreviated):

```
1.  System parameters (prm_load_by_filename)
2.  Language + timezone support
3.  Error manager (er_init)
4.  Memory subsystems (area_alloc, lock-free allocators)
5.  Disk manager (disk_manager_init)         ← volume scan
6.  File manager (file_manager_init)
7.  Page buffer (pgbuf_initialize)
8.  Log initialize (log_initialize)          ← crash recovery if needed
9.  Lock manager (lock_initialize)
10. System catalog (catalog_initialize)
11. Heap file manager (heap_manager_initialize)
12. Session manager (session_init)
13. Vacuum (vacuum_initialize)               ← starts daemon threads
14. PL engine JNI bridge (boot_start_pl_server — SERVER_MODE)
15. Parallel query workers (px_worker_manager_global_init — SERVER_MODE)
16. boot_Server_status = BOOT_SERVER_UP
```

> [!warning] Boot ordering is fragile
> Steps 5–8 form a strict dependency chain: page buffer needs disk/file managers to have located the volumes; log manager needs page buffer to read log pages; lock manager needs log manager for transaction context. Inserting a new subsystem init call between steps 5 and 9 that calls into the log or lock manager will cause assertion failures or deadlocks.

## Database Creation (`xboot_initialize_server`)

First-time creation adds steps before the normal sequence:
1. Create the active log volume (`log_create`).
2. Create initial data volume via `disk_manager`.
3. Allocate and format root heap files (`rootclass_hfid`, `ctid`).
4. Insert the `BOOT_DB_PARM` record.
5. Bootstrap system catalog tables.
6. Write the first checkpoint.

## Crash Recovery Entry Point

`log_initialize()` (called at step 8) detects a crash by comparing the log header's `chkpt_lsa` with the actual EOF LSA. If they differ, it invokes `log_recovery()` — see [[components/recovery|recovery]] for the three ARIES phases.

## Shutdown Sequence

`boot_shutdown_server()` reverses the init order:
1. Mark server status `BOOT_SERVER_DOWN` (prevents new connections).
2. Signal vacuum master to stop; join vacuum threads.
3. Final checkpoint.
4. `log_final()` — flush and close log.
5. `pgbuf_finalize()` — flush dirty pages, free buffer.
6. `file_manager_final()`, `disk_manager_final()`.
7. `lock_finalize()`.

> [!key-insight] Vacuum stops before final checkpoint
> The vacuum daemon is stopped before the final log checkpoint to ensure vacuum's own log records are stable before the checkpoint LSA is advanced. This avoids a scenario where the checkpoint skips in-flight vacuum log writes.

## Build Mode Variants

| File | Mode | Role |
|------|------|------|
| `boot_sr.c` | `SERVER_MODE` + `SA_MODE` | Server-side boot (all subsystems) |
| `boot_cl.c` | `CS_MODE` + `SA_MODE` | Client-side connect/disconnect; database path resolution |

`SA_MODE` builds link both `boot_sr.c` and `boot_cl.c` since the client and server share the same process.

## Flush Daemons

`boot_Enabled_flush_daemons` (SERVER_MODE) controls whether the page buffer and log flush daemons are active. Set to `true` after `BOOT_SERVER_UP`; set to `false` on shutdown before final flush.

## Related

- Parent: [[components/transaction|transaction]]
- [[components/log-manager|log-manager]] — `log_initialize()` called at boot step 8; triggers recovery
- [[components/recovery|recovery]] — ARIES crash recovery invoked by `log_initialize`
- [[components/storage|storage]] — disk/file/page-buffer inited before log
- [[components/vacuum|vacuum]] — started after full log initialization; stopped before shutdown flush
- [[Build Modes (SERVER SA CS)]] — `boot_sr.c` vs `boot_cl.c` split
- Source: [[sources/cubrid-src-transaction]]
