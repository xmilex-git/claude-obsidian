---
type: component
parent_module: "[[modules/src|src]]"
path: "src/cm_common/"
status: developing
purpose: "Utility layer shared between the CUBRID engine and the CUBRID Manager Java GUI (cubridmanager/ submodule): name-value IPC protocol, dynamic strings, host/process/broker statistics, broker administration wrappers, SA-mode helper executables, and JNI-to-broker bridge"
key_files:
  - "cm_dep.h (master IPC header: nvplist/dstring protocol + cm_ts_* DB task stubs + run_child + log utilities)"
  - "cm_stat.h (public API for cmstat shared lib: host/proc/broker/db stats, broker admin, CAS info)"
  - "cm_defines.h (CUBRID_DATABASE_TXT, CUBRID_ENV, FREE_MEM, REALLOC macros)"
  - "cm_utils.h/c (time_to_str, string_tokenize, run_child, cm_util_log_write_*, make_temp_filename)"
  - "cm_portable.h (Win32 shims: snprintf→_snprintf, getpid→_getpid, PATH_MAX)"
  - "cm_errmsg.h/c (cm_errors[] string table, cm_set_error, cm_err_buf_reset)"
  - "cm_dstring.c (dstring dynamic-string: dst_create/append/reset/destroy)"
  - "cm_nameval.c (nvplist key-value list: nv_create/add_nvp/get_val/writeto/readfrom)"
  - "cm_dep_tasks.c (cm_ts_* DB tasks: class info, user management, trigger info, diag monitor — SA mode)"
  - "cm_mem_cpu_stat.c (cm_get_db/broker/host/proc_stat — reads /proc or platform equivalents)"
  - "cm_broker_admin.c (cm_broker_env_start/stop, cm_broker_on/off, cm_get_broker_conf — wraps broker_admin_so.h)"
  - "cm_broker_jni.c (JNI bridge exposing cmstat functions to Java GUI; compiled as C)"
  - "cm_class_info_sa.c (cub_jobsa binary: SA-mode class info query, runs as subprocess)"
  - "cm_trigger_info_sa.c (cub_sainfo binary: SA-mode trigger query, runs as subprocess)"
  - "cm_execute_sa.h (EMS_SA_* opcode constants shared by SA helper binaries)"
tags:
  - component
  - cubrid
  - cm-common
  - manager
  - ipc
  - broker
  - statistics
related:
  - "[[modules/cm_common|cm_common module]]"
  - "[[modules/cubridmanager|cubridmanager submodule]]"
  - "[[components/broker-impl|broker-impl]]"
  - "[[components/broker-shm|broker-shm]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/cm_common/` — CUBRID Manager Shared Utilities

This directory is the C-side bridge between the CUBRID engine and the CUBRID Manager Java GUI (hosted in the `cubridmanager/` git submodule). It produces **three build artifacts** (see `cm_common/CMakeLists.txt`) and two helper binaries, all consumed at runtime by the Manager server.

## Naming Disambiguation

> [!note] Two distinct things share the name `cm_common`
> - **`src/cm_common/`** (this page) — the *implementation* directory: 12 `.c` files, 7 headers.
> - **`[[modules/cm_common|cm_common]]`** — the *top-level CMake target directory* (`/cm_common/`) that contains `CMakeLists.txt` and drives the build of the shared libraries below. It references `src/cm_common/` via the `CM_COMMON_DIR` CMake variable.

## Build Artifacts

```
cm_common/CMakeLists.txt
        │
        ├── libcmstat.so   ← CMSTAT_SOURCES
        │     cm_mem_cpu_stat.c   (host/proc/broker/db statistics)
        │     cm_broker_admin.c   (broker start/stop/on/off — wraps brokeradmin)
        │     cm_errmsg.c         (error string table)
        │     cm_utils.c          (run_child, log writers, time helpers)
        │     cm_broker_jni.c     (JNI entry points → Java GUI)
        │
        ├── libcmdep.so    ← CMDEP_SOURCES
        │     cm_dep_tasks.c      (cm_ts_* DB tasks in SA mode)
        │     cm_dstring.c        (dstring dynamic string)
        │     cm_nameval.c        (nvplist key-value list / IPC wire format)
        │     cm_utils.c          (shared)
        │
        ├── cub_jobsa      ← cm_class_info_sa.c  (SA-mode class-info subprocess)
        └── cub_sainfo     ← cm_trigger_info_sa.c (SA-mode trigger-info subprocess)
```

## Subsystems

### 1. Name-Value IPC Protocol (`dstring` + `nvplist`)

The original CUBRID Manager wire format is a flat name-value list encoded as colon-delimited text:

```
name:value\n
name:value\n
```

**`dstring`** (`cm_dstring.c`) is a grow-on-demand heap string (`dst_create`, `dst_append`, `dst_destroy`). It underlies every `nvpair`.

**`nvplist`** (`cm_nameval.c`) is a resizable array of `nvpair*`, with configurable delimiters (`":"`) and end-markers (`"\n"`). Key API:

| Function | Purpose |
|----------|---------|
| `nv_create(defsize, lom, lcm, dm, em)` | Allocate list with open/close/delim/end markers |
| `nv_add_nvp(ref, name, value)` | Append a string pair |
| `nv_add_nvp_int / _int64 / _float / _time` | Typed variants |
| `nv_get_val(ref, name)` | Linear scan lookup by name |
| `nv_update_val(ref, name, value)` | In-place update |
| `nv_writeto / nv_readfrom(ref, filename)` | Serialise/deserialise to/from file |
| `nv_locate(ref, marker, index, ilen)` | Find section marker (list opener) |

Both `libcmstat.so` and `libcmdep.so` link this protocol; the Java GUI sends requests and reads responses over it.

### 2. Database & System Statistics (`cm_stat.h` / `cm_mem_cpu_stat.c`)

`libcmstat.so` exposes a unified statistics API covering host, process, broker, and database execution counters. It reads from OS-level sources (Linux `/proc`, AIX `libperfstat`, Windows `psapi`) and from the broker shared-memory segment.

Key types and functions:

| Type/Function | Description |
|---------------|-------------|
| `T_CM_HOST_STAT` | CPU user/kernel/idle/iowait + physical/swap memory |
| `T_CM_PROC_STAT` | Per-PID: cpu_user, cpu_kernel, mem_physical, mem_virtual |
| `T_CM_DB_PROC_STAT` / `*_ALL` | DB process stats by name |
| `T_CM_BROKER_PROC_STAT` / `*_ALL` | Broker + per-CAS stats |
| `T_CM_DB_EXEC_STAT` | ~130 `uint64_t` counters: file I/O, page buffer (LRU zones), log, lock, transactions, B-tree ops, heap ops, query manager, sort, plan cache, vacuum, HA replication delay |
| `T_CM_CAS_INFO` / `T_CM_BROKER_INFO` | Live CAS/broker runtime info |
| `cm_get_db_exec_stat(db_name, ...)` | Entry point: reads engine execution counters |
| `cm_get_host_stat(...)` | Host CPU/memory snapshot |
| `cm_get_cas_info(br_name, ...)` | CAS runtime info for a broker |

The `T_CM_DB_EXEC_STAT` struct mirrors `src/base/perf_monitor.c` counter layout — **it must stay in sync** with the server-side performance monitor.

### 3. Broker Administration (`cm_broker_admin.c`)

Thin wrappers around the `brokeradmin` shared library (`broker_admin_so.h`). Loaded dynamically at runtime (via `dlopen` on POSIX, `LoadLibrary` on Win32). All functions take a `T_CM_ERROR *err_buf` for error reporting.

| Function | Wraps |
|----------|-------|
| `cm_broker_env_start / stop` | `uc_start / uc_stop` |
| `cm_broker_on / off` | `uc_on / uc_off` |
| `cm_broker_as_restart` | `uc_restart` |
| `cm_get_broker_conf` | reads `cubrid_broker.conf` / `unicas.conf` |
| `cm_get_broker_info` / `cm_get_cas_info` | broker + CAS live info |

Four broker config files are registered by ID (`T_UNICAS_FILE_ID`): admin log, `unicas.conf`, `cubrid_cas.conf`, `cubrid_broker.conf`.

### 4. SA-Mode DB Task Engine (`cm_dep_tasks.c`)

Implements `cm_ts_*` functions declared in `cm_dep.h`. These run in **SA mode** (standalone, engine embedded in-process) to handle requests from the Manager GUI that require direct DB access without a running server:

- `cm_tsDBMTUserLogin` — authenticate against `databases.txt`
- `cm_ts_class_info` / `cm_ts_class` — schema class enumeration
- `cm_ts_get_triggerinfo` — trigger list
- `cm_ts_update_attribute` / `cm_ts_userinfo` / `cm_ts_create_user / update_user / delete_user` — schema management
- `cm_ts_optimizedb` — trigger index statistics update
- `uDatabaseMode(dbname, &ha_mode)` — returns `T_DB_SERVICE_MODE` (NONE/SA/CS)
- `uIsDatabaseActive(dbname)` — checks for running server lock file

Imports engine headers directly: `dbi.h`, `perf_monitor.h`, `numeric_opfunc.h`, `intl_support.h`, `system_parameter.h`, `dbtype.h`.

### 5. SA-Mode Helper Binaries (`cub_jobsa`, `cub_sainfo`)

Two small standalone executables launched as **subprocesses** by `libcmdep.so`. They link `cubridsa` (SA-mode engine library), open the database in standalone mode, and write results to a temp file read back by the parent.

- **`cub_jobsa`** (`cm_class_info_sa.c`) — class info + DBMT user login; opcodes `EMS_SA_CLASS_INFO (1)` / `EMS_SA_DBMT_USER_LOGIN (2)`
- **`cub_sainfo`** (`cm_trigger_info_sa.c`) — trigger list dump

The subprocess pattern allows the Manager to query a stopped (offline) database without starting a full server.

### 6. JNI Bridge (`cm_broker_jni.c`)

Compiled as **C** (not C++, hence `SET_SOURCE_FILES_PROPERTIES ... LANGUAGE C`). Exposes `libcmstat.so` functions as JNI entry points callable from the Java GUI. Uses `SUPPORT_BROKER_JNI` compile guard and provides `FAIL_RET` / `FAIL_RET_X` macros for checked JNI call chains.

### 7. Utilities, Portability & Error Messages

| File | Notes |
|------|-------|
| `cm_portable.h` | Win32 shims: `snprintf → _snprintf`, `getpid → _getpid`, `strcasecmp → _stricmp`, `PATH_MAX 256`, `CUB_MAXHOSTNAMELEN 256` |
| `cm_defines.h` | `CUBRID_DATABASE_TXT "databases.txt"`, `CUBRID_ENV`, `FREE_MEM(PTR)` null-safe free macro, `REALLOC` |
| `cm_utils.c` | `run_child(argv, wait_flag, stdin/stdout/stderr_file, exit_status)` — POSIX `fork/exec` or Win32 `CreateProcess`; `cm_util_log_write_result/errid/errstr/command`; `time_to_str`; `make_temp_filename/filepath` |
| `cm_errmsg.c` | `cm_errors[]` — 9-entry string table indexed by `(err_code - CM_MIN_ERROR)`. Codes `2000–2008` (distinct from engine's `ERR_*` codes in `cm_dep.h`, which start at 1000). |

## Error Code Systems (two independent ranges)

`cm_common` uses **two independent error code sets** — not the engine's `error_code.h`:

| Range | Location | Used by |
|-------|----------|---------|
| `ERR_NO_ERROR (1000)` … `ERR_WARNING (1350)` | `cm_dep.h` | `libcmdep.so` / `cm_ts_*` tasks |
| `CM_UNKOWN_ERROR (2000)` … `CM_READ_STATDUMP_INFO_ERROR (2008)` | `cm_stat.h` | `libcmstat.so` / stat API |
| `CM_NO_ERROR = 0` | `cm_dep.h` | log result helper |

## Dependency Graph

```
cm_dep_tasks.c ──────────────────────────────────────────────────────┐
  includes: dbi.h, perf_monitor.h, numeric_opfunc.h, intl_support.h  │
            system_parameter.h, dbtype.h, object_primitive.h          │
            (all from src/base/ or src/compat/)                       │
                                                                      ▼
cm_broker_admin.c → broker_admin_so.h (dlopen at runtime → brokeradmin lib)
cm_mem_cpu_stat.c → /proc (Linux) | libperfstat (AIX) | psapi (Windows)
cm_broker_jni.c   → jni.h (JDK)
cm_class_info_sa.c → dbi.h + cubridsa (SA-mode engine)
cm_trigger_info_sa.c → dbi.h + cubridsa (SA-mode engine)
```

The key boundary: `cm_dep_tasks.c` includes engine headers directly and thus belongs to `libcmdep.so` (not `libcmstat.so`), while `libcmstat.so` only reads OS-level stats and the broker shared memory, keeping engine headers out.

## Relation to `cubridmanager/` Submodule

The `cubridmanager/` git submodule contains the CUBRID Manager server (Java). It calls into `libcmstat.so` via JNI (`cm_broker_jni.c`) for live statistics and broker management, and spawns `cub_jobsa` / `cub_sainfo` subprocesses for offline DB metadata queries. The nvplist text protocol in `libcmdep.so` is also the wire format between the Manager GUI client and Manager server.

## Quick Reference

| Task | File |
|------|------|
| Parse broker conf | `cm_broker_admin.c` → `cm_get_broker_conf` |
| Get host CPU/mem | `cm_mem_cpu_stat.c` → `cm_get_host_stat` |
| Get DB exec counters | `cm_mem_cpu_stat.c` → `cm_get_db_exec_stat` |
| Spawn child process | `cm_utils.c` → `run_child` |
| Log error to file | `cm_utils.c` → `cm_util_log_write_errstr` |
| Build nvplist response | `cm_nameval.c` → `nv_create` + `nv_add_nvp_*` |
| Query offline DB class info | `cub_jobsa` (subprocess) |
| Query offline DB triggers | `cub_sainfo` (subprocess) |
| Restart a CAS worker | `cm_broker_admin.c` → `cm_broker_as_restart` |

## Related

- Parent module: [[modules/src|src]]
- CMake target owner: [[modules/cm_common|cm_common module]]
- Consumer: [[modules/cubridmanager|cubridmanager submodule]]
- Broker admin lib: [[components/broker-impl|broker-impl]] (provides `brokeradmin`)
- Broker SHM: [[components/broker-shm|broker-shm]] (stats read from shared memory)
- [[Build Modes (SERVER SA CS)]] — SA mode is central to `cm_dep_tasks.c` and the SA helper binaries
- Source: [[sources/cubrid-src-cm-common]]
