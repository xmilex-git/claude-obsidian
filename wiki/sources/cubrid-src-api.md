---
type: source
title: "CUBRID src/api/ — Public C API Extensions"
source_path: "src/api/"
ingested: 2026-04-23
status: complete
files_read:
  - "cubrid_log.h (full)"
  - "cubrid_log.c (full, ~2000 lines)"
pages_created:
  - "[[components/api|api]]"
  - "[[components/cubrid-log-cdc|cubrid-log-cdc]]"
tags:
  - source
  - cubrid
  - cdc
  - api
related:
  - "[[components/api|api]]"
  - "[[components/cubrid-log-cdc|cubrid-log-cdc]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/transaction|transaction]]"
---

# Source: `src/api/` — Public C API Extensions

## Directory Contents

`src/api/` contains two files:

| File | Purpose |
|------|---------|
| `cubrid_log.h` | CDC public header: types, error codes, function declarations |
| `cubrid_log.c` | CDC implementation (~1990 lines, `CS_MODE` only) |

No `AGENTS.md` is present in this directory. The top-level `CUBRID/AGENTS.md` identifies it as "Public C API extensions (cubrid_log.c)".

## Key Findings

### Architecture

The CDC API is a standalone client-side module. It does not go through the broker/CAS pipeline. Instead `cubrid_log_connect_server` opens a raw `CSS_CONN_ENTRY` directly to `cub_server` and speaks three CDC-specific network requests:

- `NET_SERVER_CDC_START_SESSION` — configure filter and open session
- `NET_SERVER_CDC_FIND_LSA` — resolve a Unix timestamp to a WAL LSA
- `NET_SERVER_CDC_GET_LOGINFO_METADATA` — fetch item count + byte length for the next batch
- `NET_SERVER_CDC_GET_LOGINFO` — fetch the actual packed log data
- `NET_SERVER_CDC_END_SESSION` — close session

### State Machine

Four stages (`CONFIGURATION → PREPARATION → EXTRACTION → CONFIGURATION`) enforced by a file-scope `CUBRID_LOG_STAGE g_stage` global. Every public function checks the current stage and returns `CUBRID_LOG_INVALID_FUNC_CALL_STAGE` if called out of order.

### Memory Model

Two persistent realloc'd buffers:
- `g_log_infos` — raw packed bytes received from server
- `g_log_items` — array of `CUBRID_LOG_ITEM` structs

DML items additionally allocate per-call arrays for column indexes and data pointers. These are freed by `cubrid_log_clear_log_item`. Column data pointers themselves are zero-copy pointers into `g_log_infos`.

### `supplemental_log` Prerequisite

The server parameter `supplemental_log` must be enabled. When off, `NET_SERVER_CDC_START_SESSION` returns `ER_CDC_NOT_AVAILABLE` and the client surfaces `CUBRID_LOG_UNAVAILABLE_CDC_SERVER (-34)`. This is the primary server-side gate.

### DBA Requirement

`cubrid_log_connect_server` performs a temporary `db_restart` + `au_login` + `au_is_dba_group_member` check before opening the CDC connection. Non-DBA users get `CUBRID_LOG_FAILED_LOGIN (-33)`.

### Not Thread-Safe

All state is in file-scope globals. Single-threaded consumer design. Multi-session CDC requires multiple processes.

## Pages Created

- [[components/api|api]] — hub for `src/api/`, CS_MODE-only constraint, design intent
- [[components/cubrid-log-cdc|cubrid-log-cdc]] — full CDC API: four-phase model, all functions, data types, error codes, usage pattern, threading model, `supplemental_log` dependency
