---
type: meta
title: "Hot Cache"
updated: 2026-04-23T21:00:00
tags:
  - meta
  - hot
status: active
---

# Recent Context

## Last Updated
2026-04-23. CUBRID ingest rounds 1+2 complete. 38 component pages. Parallel-agent pipeline working.

## Key Recent Facts (top of mind)

### From src/query/parallel
- Single global named worker pool (`"parallel-query"`); per-query `worker_manager` reserves N slots via lock-free CAS.
- Auto-degree: `floor(log2(pages/threshold)) + 2`, capped by core count.
- Worker errors live thread-local; must be moved to `err_messages_with_lock` before exit.
- See [[components/parallel-query]].

### From src/query
- `qexec_execute_mainblock` (`query_executor.c`) ~27K lines — single dispatch for SELECT, all DML, set ops, CONNECT BY, MERGE.
- `SCAN_ID` is polymorphic over **15** scan types, unified via `scan_next_scan()`.
- Hash GROUP BY runtime spill: 2000-tuple calibration window + 50% selectivity threshold.
- `filter_pred_cache` uses exclusive-lease (no shared locks).
- See [[components/query]], [[components/query-executor]], [[components/scan-manager]].

### From src/parser
- `PT_NODE` is the universal parse tree node — tagged union (`info` member chosen by `node_type`).
- Function tables for `parser_new_node`/`init_node`/`print_tree` are **ordinal-indexed by `PT_NODE_TYPE`** — silent crash if you misorder.
- Bison: `YYMAXDEPTH 1000000`, `container_2..11` helper structs (workaround for single-`$$` constraint at 646 KB).
- `parser_block_allocator::dealloc` is a **no-op** — entire arena freed at `parser_free_parser`.
- See [[components/parser]], [[components/parse-tree]], [[components/xasl-generation]].

### From src/storage
- Buffer pool uses **3-zone LRU** (hot/buffer/victim); only zone 3 evictable. Vacuum workers excluded from hot promotion.
- DWB recovery runs **before** WAL redo to fix torn writes at the physical layer.
- WAL ordering enforced **inside `pgbuf_flush_with_wal`** (not in callers).
- LOB external delete uses `LOG_POSTPONE` (not WAL) → bugs typically in postpone machinery, not `es.c`.
- B-tree dispatch parameterized by **18 `btree_op_purpose` values** — explains why `btree.c` is 37 K lines.
- See [[components/storage]], [[components/page-buffer]], [[components/btree]].

### Cross-cutting recurring patterns
- [[Memory Management Conventions]]: arena allocators (parser_alloc, db_private_alloc) + `free_and_init` + `memory_wrapper.hpp` last-include.
- [[Build Modes (SERVER SA CS)]]: `#error Belongs to server module` is the standard guard.
- [[Code Style Conventions]]: `// *INDENT-OFF*` markers around macros; 120-col GNU braces.

## Recent Changes
- 38 component pages now (from initial 4).
- 6 new source summaries: [[cubrid-AGENTS]], [[cubrid-src-query-parallel]], [[cubrid-src-query]], [[cubrid-src-parser]], [[cubrid-src-storage]] — full ingest log in [[log]].
- Sub-indexes auto-updated: [[components/_index]], [[sources/_index]].

## Active Threads
- **Round 3 next**: `src/transaction`, `src/object`, `src/base`, `src/xasl`, then `src/compat`, `src/sp`, `src/thread`, `src/connection`, then `src/broker` (impl), `src/communication`, `src/method`, `src/loaddb`, then smaller dirs (`executables`, `monitor`, `session`, `cm_common`, `api`, `debugging`, `win_tools`, `heaplayers`).
- Open follow-ups (from agent reports):
  - Flow page: `pgbuf_fix → dwb_add_page → fileio_write` (page write lifecycle + WAL ordering)
  - Flow page: B-tree insert with MVCC
  - Flow page: `query-compile-flow` (one SELECT through all 6 parser passes)
  - Component page: `slotted_page` (referenced by heap + btree)
  - Component page: `vacuum` (in `src/transaction/`)
  - Ingest `src/optimizer/` (still stub) and `src/xasl/` (current xasl-generation refs depend on it)
- Obsidian Git auto-push every 30 min → branch `cubrid1` on `xmilex-git/claude-obsidian`.
