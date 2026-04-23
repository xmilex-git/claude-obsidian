---
type: source
title: "cubrid src/cm_common/"
date: 2026-04-23
tags:
  - source
  - cubrid
  - cm-common
  - manager
  - statistics
  - broker
status: ingested
related:
  - "[[components/cm-common-src]]"
  - "[[modules/cm_common|cm_common module]]"
  - "[[modules/cubridmanager|cubridmanager submodule]]"
---

# Source: `src/cm_common/`

**Ingested:** 2026-04-23
**Files read:** `cm_defines.h`, `cm_portable.h`, `cm_utils.h`, `cm_dep.h`, `cm_stat.h`, `cm_errmsg.h`, `cm_execute_sa.h`, `cm_utils.c`, `cm_dstring.c`, `cm_nameval.c`, `cm_errmsg.c`, `cm_broker_admin.c`, `cm_mem_cpu_stat.c` (header), `cm_dep_tasks.c` (header), `cm_class_info_sa.c` (header), `cm_trigger_info_sa.c` (header), `cm_broker_jni.c` (header), plus `cm_common/CMakeLists.txt`

## What This Directory Is

`src/cm_common/` is the C implementation of utilities shared between the CUBRID engine and the CUBRID Manager Java GUI. It builds three artifacts via `cm_common/CMakeLists.txt`:

- **`libcmstat.so`** — host/process/broker/DB statistics + broker admin + JNI bridge (public header: `cm_stat.h`)
- **`libcmdep.so`** — nvplist IPC protocol + SA-mode DB tasks (public header: `cm_dep.h`)
- **`cub_jobsa`** / **`cub_sainfo`** — subprocess binaries for offline database queries

## Key Discovery: The Naming Split

The top-level `cm_common/` directory (CMake target owner) is distinct from `src/cm_common/` (implementation). The CMake variable `CM_COMMON_DIR` points from the former to the latter. This is the same split as `broker/` (CMake target) vs `src/broker/` (impl).

## Key APIs Discovered

- `nvplist` / `dstring` — the Manager's text IPC wire format; colon-delimited key:value\n pairs
- `T_CM_DB_EXEC_STAT` — 130+ `uint64_t` counters mirroring `perf_monitor.c`; must stay in sync
- `T_CM_HOST_STAT`, `T_CM_PROC_STAT`, `T_CM_BROKER_PROC_STAT` — OS-level resource stats
- `run_child()` — cross-platform subprocess launcher (fork/exec or CreateProcess)
- `cm_ts_*` task functions — SA-mode DB operations (class/trigger/user info) in `libcmdep.so`
- `cm_broker_jni.c` — compiled as C (not C++); JNI bridge from Java Manager to `libcmstat.so`
- `EMS_SA_CLASS_INFO (1)` / `EMS_SA_DBMT_USER_LOGIN (2)` — opcode protocol for SA helper binaries

## Pages Created

- [[components/cm-common-src]] — full component hub page
