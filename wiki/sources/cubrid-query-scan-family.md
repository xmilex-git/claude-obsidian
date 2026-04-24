---
created: 2026-04-23
type: source
title: "CUBRID query/ SCAN FAMILY deep dive"
date: 2026-04-23
status: processed
files_studied:
  - "src/query/query_hash_scan.{c,h}"
  - "src/query/query_hash_join.{c,h}"
  - "src/query/set_scan.{c,h}"
  - "src/query/show_scan.{c,h}"
  - "src/query/scan_json_table.{cpp,hpp}"
  - "src/query/partition.h"
  - "src/query/partition_sr.h"
tags:
  - source
  - cubrid
  - query
  - scan
related:
  - "[[components/scan-hash|scan-hash]]"
  - "[[components/scan-set|scan-set]]"
  - "[[components/scan-show|scan-show]]"
  - "[[components/scan-json-table|scan-json-table]]"
  - "[[components/partition-pruning|partition-pruning]]"
  - "[[components/scan-manager|scan-manager]]"
---

# CUBRID Query Layer — Scan Family Deep Dive

Granular deep-dive into five scan-type implementations under `src/query/`, producing one wiki page per scan type.

## Files Studied

| File | Lines | Notes |
|---|---|---|
| `query_hash_scan.h` | 184 | `HASH_METHOD`, `HASH_LIST_SCAN`, `HASH_SCAN_KEY`, `HASH_SCAN_VALUE`, `FHSID`, `TFTID`, `fhs_*` API |
| `query_hash_scan.c` | ~700 | Key/value alloc, `qdata_hash_scan_key`, `fhs_*` extendible hash implementation |
| `query_hash_join.h` | 421 | `HASHJOIN_STATUS`, `HASHJOIN_MANAGER`, `HASHJOIN_CONTEXT`, all join struct types |
| `query_hash_join.c` | ~1800 | `qexec_hash_join`, `hjoin_*` build/probe/partition/parallel |
| `set_scan.h` | 33 | Single prototype |
| `set_scan.c` | 158 | `qproc_next_set_scan` — the entire set scan |
| `show_scan.h` | 47 | `SHOWSTMT_ARRAY_CONTEXT`, `thread_start_scan` |
| `show_scan.c` | ~700 | `showstmt_scan_init`, dispatch table, `thread_scan_mapfunc` |
| `scan_json_table.hpp` | 173 | `cubscan::json_table::scanner` class |
| `scan_json_table.cpp` | ~560 | `scanner::cursor`, `scan_next_internal`, `open`, `clear` |
| `partition.h` | 28 | `MAX_PARTITIONS = 1024` |
| `partition_sr.h` | 141 | `PRUNING_CONTEXT`, `PRUNING_SCAN_CACHE`, full API |

## Pages Created / Upgraded

- **Created** [[components/scan-hash|scan-hash]] — `query_hash_scan` + `query_hash_join`: serial hash scan, three HASH_METHOD variants, FHS extendible hash, build/probe lifecycle, in-memory vs partition decision
- **Created** [[components/scan-set|scan-set]] — `set_scan`: F_SEQUENCE vs DB_SET iteration, order guarantees, per-element cost
- **Created** [[components/scan-show|scan-show]] — `show_scan`: static dispatch table, 15+ registered SHOW types, array vs streaming patterns, DBA gating
- **Created** [[components/scan-json-table|scan-json-table]] — `scan_json_table`: SQL:2016 JSON_TABLE, depth-first cursor, RapidJSON path extraction, sibling isolation
- **Upgraded** [[components/partition-pruning|partition-pruning]] — added SELECT vs DML pruning differences, `partition_prune_spec` algorithm detail, partition cache ownership rules, `PRUNING_SCAN_CACHE` multi-row DML reuse, aggregate O(1) MIN/MAX, 2-phase (static+dynamic) pruning

## Key Findings

1. Hash join uses a 3-way execution path (SINGLE / PARTITION / PARALLEL) decided upfront by `hjoin_check_partition` based on estimated memory vs `PRM_ID_MAX_HASH_LIST_SCAN_SIZE`; there is no runtime mid-build spill.
2. Set scan is the simplest scan implementation: 158 lines, one function, two code paths (F_SEQUENCE linked-list vs DB_SET indexed access).
3. SHOW scan uses a static `show_Requests[]` jump table; adding a new SHOW statement requires one entry in `showstmt_scan_init()` plus a matching `SHOWSTMT_METADATA` entry in show-meta.
4. JSON_TABLE scanner uses a depth-first cursor array (`cursor[tree_height]`) with a `m_scan_cursor_depth` tracker; RapidJSON path compilation is repeated per row (no cache).
5. Partition pruning operates in 2 phases: static (client/optimizer) and dynamic (server/`partition_prune_spec`); the `PRUNING_SCAN_CACHE` linked list avoids redundant `HEAP_SCANCACHE` opens across multi-row DML.
