---
type: component
parent_module: "[[components/base|base]]"
path: "src/base/"
status: developing
purpose: "Centralized runtime configuration: ~400 PRM_ID_* parameters read from cubrid.conf, accessible via prm_get_*_value() throughout all modules"
key_files:
  - "system_parameter.c/h (largest file in src/base; prm_Def[] array, prm_get_*_value API)"
  - "databases_file.c/h (databases.txt parser and database registry)"
  - "ini_parser.c/h (INI file parser used for broker config)"
  - "environment_variable.c/h ($CUBRID path resolution)"
tags:
  - component
  - cubrid
  - configuration
  - system-parameter
related:
  - "[[components/base|base]]"
  - "[[components/error-manager|error-manager]]"
  - "[[components/porting|porting]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/page-buffer|page-buffer]]"
created: 2026-04-23
updated: 2026-04-23
---

# `system_parameter` — System Configuration Subsystem

The `system_parameter` module is the single source of truth for all CUBRID runtime configuration. It defines ~400 `PRM_ID_*` enum values, reads `cubrid.conf`, and exposes typed accessors used throughout every engine module.

> [!key-insight] Largest file in src/base/
> `system_parameter.c` is the largest file in the `base/` directory. Its `prm_Def[]` array contains one entry per parameter with name, type, default, range, and access flags.

## Parameter ID Enum (`param_id` in `system_parameter.h`)

```c
enum param_id {
  PRM_FIRST_ID = 0,
  PRM_ID_ER_LOG_DEBUG = 0,       // bool: enable debug log
  PRM_ID_ER_LOG_LEVEL,           // int: error log level
  PRM_ID_PB_NBUFFERS,            // int: page buffer count
  PRM_ID_PAGE_BUFFER_SIZE,       // int: page buffer size in bytes
  PRM_ID_LK_TIMEOUT_SECS,        // int: lock timeout seconds
  PRM_ID_LK_ESCALATION_AT,       // int: lock escalation threshold
  PRM_ID_LOG_CHECKPOINT_INTERVAL, // int: checkpoint interval
  PRM_ID_XASL_CACHE_MAX_ENTRIES, // int: XASL plan cache size
  PRM_ID_PARALLELISM,            // int: max parallel query degree
  PRM_ID_MAX_PARALLEL_WORKERS,   // int: parallel worker pool cap
  PRM_ID_PARALLEL_HEAP_SCAN_PAGE_THRESHOLD,  // int: page threshold for parallel heap scan
  PRM_ID_PARALLEL_HASH_JOIN_PAGE_THRESHOLD,  // int: page threshold for parallel hash join
  PRM_ID_PARALLEL_SORT_PAGE_THRESHOLD,       // int: page threshold for parallel sort
  // ... ~400 total
  PRM_LAST_ID
};
```

Parameters span all subsystems: buffer pool, locking, logging, query optimization, HA, parallel execution, session, networking, and more.

## Access API

```c
// Read by type — most common pattern throughout the engine
bool        prm_get_bool_value(PARAM_ID id);
int         prm_get_integer_value(PARAM_ID id);
float       prm_get_float_value(PARAM_ID id);
const char *prm_get_string_value(PARAM_ID id);
int        *prm_get_integer_list_value(PARAM_ID id);
UINT64      prm_get_bigint_value(PARAM_ID id);
```

Usage pattern found throughout the engine:
```c
int page_threshold = prm_get_integer_value(PRM_ID_PARALLEL_HEAP_SCAN_PAGE_THRESHOLD);
bool log_debug    = prm_get_bool_value(PRM_ID_ER_LOG_DEBUG);
int pb_size       = prm_get_integer_value(PRM_ID_PAGE_BUFFER_SIZE);
```

## Parameter Error Codes (`SYSPRM_ERR`)

```c
typedef enum {
  PRM_ERR_NO_ERROR = NO_ERROR,
  PRM_ERR_UNKNOWN_PARAM = 12,
  PRM_ERR_BAD_VALUE = 13,
  PRM_ERR_CANNOT_CHANGE = 24,
  PRM_ERR_NOT_FOR_CLIENT = 25,
  PRM_ERR_NOT_FOR_SERVER = 26,
  // ... 34 error variants
} SYSPRM_ERR;
```

Some parameters are server-only, some client-only. Attempts to set server parameters from a client session return `PRM_ERR_NOT_FOR_CLIENT`.

## Compatibility Modes

```c
enum compat_mode {
  COMPAT_CUBRID,   // default CUBRID behavior
  COMPAT_MYSQL,    // MySQL compatibility (PRM_ID_COMPAT_MODE)
  COMPAT_ORACLE    // Oracle compatibility
};
```

Controlled by `PRM_ID_COMPAT_MODE`. Affects SQL behavior for empty strings, outer joins, `NULL` handling, etc.

## Configuration Sources (priority order)

1. `cubrid.conf` — parsed by `system_parameter.c` on server/client startup
2. Environment variables — `$CUBRID`, `$CUBRID_DATABASES`, etc.
3. Per-session `SET SYSTEM PARAMETERS` — some parameters are session-changeable
4. CLI flags — `cubrid server start --parameter value`

## Adding a New Parameter

> [!warning] Required steps for new parameters
> 1. Add `PRM_ID_*` to `param_id` enum **before** `PRM_LAST_ID`
> 2. Add an entry to the `prm_Def[]` array in `system_parameter.c` (name, type, default, range, flags)
> 3. Update `PRM_LAST_ID` if needed
> 4. Add to `cubrid.conf.default` template if it should appear in default config
> 5. Update relevant message catalog if parameter has user-visible description

Parameter order in the enum must match position in `prm_Def[]` — they are indexed positionally.

## Key Parameter Families

| Family | PRM_ID prefix | What it controls |
|--------|---------------|-----------------|
| Error logging | `ER_` | Log level, log file path, debug mode |
| Page buffer | `PB_` | Buffer pool size, flush ratios, LRU tuning |
| Locking | `LK_` | Timeout, escalation threshold, deadlock interval |
| Logging/WAL | `LOG_` | Checkpoint interval, buffer size, isolation level |
| Query cache | `XASL_CACHE_` | Plan cache entries, clones, timeout |
| Parallel query | `PARALLEL_` | Degree, worker cap, per-type page thresholds |
| HA | `HA_` | Replication mode, server state, node list |
| Session | various | Auto-commit, timeout, isolation |

## Cross-Module Visibility

`system_parameter.h` includes `error_manager.h` and `porting.h`, making it one of the higher-level headers in `base/`. Most engine modules include it directly to read their configuration.

## Related

- [[components/base|base]] — parent hub
- [[components/error-manager|error-manager]] — `PRM_ID_ER_LOG_DEBUG` controls `er_log_debug()` firing
- [[components/parallel-query|parallel-query]] — reads `PRM_ID_PARALLELISM`, `PRM_ID_MAX_PARALLEL_WORKERS`, and per-type thresholds
- [[components/page-buffer|page-buffer]] — reads `PRM_ID_PB_NBUFFERS`, `PRM_ID_PAGE_BUFFER_SIZE`, etc.
- Source: [[sources/cubrid-src-base|cubrid-src-base]]
