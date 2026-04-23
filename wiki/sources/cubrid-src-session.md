---
type: source
title: "CUBRID src/session/ — Per-connection session state"
source_path: "src/session/"
date_ingested: 2026-04-23
status: complete
files_read:
  - "session.h"
  - "session.c (full, ~3100 lines)"
tags:
  - source
  - cubrid
  - session
  - server
related:
  - "[[components/session|session]]"
  - "[[components/session-state|session-state]]"
  - "[[components/session-variables|session-variables]]"
---

# Source: `src/session/`

Ingested 2026-04-23. Directory contains two files: `session.h` (public API) and `session.c` (~3100 lines, internal impl). No AGENTS.md present.

## What Was Found

- `SESSION_STATE` struct (defined in `session.c` only, not the header) — 20+ fields covering identity, insert-ID tracking, row count, user variables, prepared statements, holdable cursor results, session parameter overrides, query trace, timezone, private LRU index, bulk-load session, PL engine session.
- `ACTIVE_SESSIONS` global singleton holding a `cubthread::lockfree_hashmap<SESSION_ID, session_state>` (1000 buckets).
- Full lifecycle: `session_states_init` → `session_state_create` → `session_check_session` (per-request refresh) → `session_state_destroy` (on disconnect or timeout).
- Session-control daemon (60-second reaper): `session_remove_expired_sessions()` with a two-pass buffer-then-erase algorithm to avoid corrupting the lock-free iterator.
- Prepared statement cache: up to 20 per session, keyed by name; XASL plan looked up via `xcache_find_sha1()` — session metadata is local, compiled plan is server-global.
- Holdable cursor store: `SESSION_QUERY_ENTRY` list with `session_preserve_temporary_files()` to prevent temp file cleanup at transaction end.
- Session variable store: up to 20 `@var` bindings per session, with deep copy + two magic side effects (`collect_exec_stats`, `trace_plan`).
- System parameter overrides: flat `SESSION_PARAM *` array looked up via `prm_Def_session_idx[id]` in O(1).
- Private LRU partition: each session owns its own page-buffer LRU zone (`pgbuf_assign_private_lru`).
- `PL_SESSION *pl_session_p`: Java PL engine context, allocated on session create, deleted on session uninit.

## Key Insights

1. **Session vs transaction**: session state outlives any individual transaction; `@vars` and session parameters are never rolled back.
2. **Zero-hash hot path**: in SERVER_MODE `session_get_session_state()` reads `thread_p->conn_entry->session_p` directly — no hashmap lookup on the per-request fast path.
3. **Prepared statement locality**: the session stores the metadata (name, alias_print, SHA-1, serialized info). The XASL cache entry is server-global and can be evicted independently.
4. **Two-pass reaper**: cannot delete while iterating the lock-free hashmap; buffers IDs, then erases.

## Pages Created

- [[components/session]] — hub page
- [[components/session-state]] — struct & lifecycle detail
- [[components/session-variables]] — @var bindings & session param overrides
