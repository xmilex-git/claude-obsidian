---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/query_hash_scan.c + query_hash_join.c"
status: active
purpose: "Serial in-memory and file-backed hash scan used as the HASH_LIST_SCAN backend; foundation layer for the query_hash_join orchestrator which upgrades to partitioned or parallel execution when data exceeds PRM_ID_MAX_HASH_LIST_SCAN_SIZE"
key_files:
  - "src/query/query_hash_scan.c — key/value alloc, hash computation, FHS (File Hash Structure) extendible-hash backend"
  - "src/query/query_hash_scan.h — HASH_METHOD enum, HASH_LIST_SCAN, HASH_SCAN_KEY, HASH_SCAN_VALUE, FHSID, TFTID"
  - "src/query/query_hash_join.c — HASHJOIN_MANAGER orchestration, build/probe phases, partitioning decision"
  - "src/query/query_hash_join.h — HASHJOIN_STATUS, HASHJOIN_MANAGER, HASHJOIN_CONTEXT, public API"
public_api:
  - "qdata_alloc_hscan_key(thread_p, val_cnt, alloc_vals) → HASH_SCAN_KEY*"
  - "qdata_free_hscan_key(thread_p, key, val_count)"
  - "qdata_alloc_hscan_value(thread_p, tpl) → HASH_SCAN_VALUE*"
  - "qdata_alloc_hscan_value_OID(thread_p, scan_id_p) → HASH_SCAN_VALUE*"
  - "qdata_free_hscan_value(thread_p, value)"
  - "qdata_free_hscan_entry(key, data, args) → int"
  - "qdata_build_hscan_key(thread_p, vd, regu_list, key) → int"
  - "qdata_hash_scan_key(key, ht_size, hash_method) → unsigned int"
  - "qdata_hscan_key_eq(key1, key2) → int"
  - "qdata_copy_hscan_key(thread_p, key, probe_regu_list, vd) → HASH_SCAN_KEY*"
  - "qdata_copy_hscan_key_without_alloc(thread_p, key, probe_regu_list, new_key) → HASH_SCAN_KEY*"
  - "fhs_create(thread_p, fhsid, exp_num_entries) → FHSID*"
  - "fhs_destroy(thread_p, fhsid) → int"
  - "fhs_insert(thread_p, fhsid, key, value_ptr) → void*"
  - "fhs_search(thread_p, hlsid, value_ptr) → EH_SEARCH"
  - "fhs_search_next(thread_p, hlsid, value_ptr) → EH_SEARCH"
  - "qexec_hash_join(thread_p, xasl, query_id, val_descr) → int"
  - "hjoin_execute(thread_p, manager, context) → int"
  - "hjoin_fetch_key(thread_p, fetch_info, tuple_record, key, compare_key, need_skip_next) → int"
tags:
  - component
  - cubrid
  - query
  - scan
  - hash
  - join
related:
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/parallel-hash-join|parallel-hash-join]]"
  - "[[components/list-file|list-file]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/query|query]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `query_hash_scan` / `query_hash_join` — Serial Hash Scan

The hash-scan layer implements build-then-probe hash join semantics using an in-memory hash table (`MHT_HLS_TABLE`) or a file-backed extendible-hash table (`FHSID`). The `query_hash_join.c` orchestrator (`qexec_hash_join`) decides which execution mode to use and delegates the low-level key/value mechanics to `query_hash_scan.c`.

## Purpose

- Provide a hash-based equi-join for `HASHJOIN_PROC_NODE` XASL nodes.
- Serve as the build/probe engine for all three execution paths: single (in-memory), partitioned (serial multi-pass), and parallel (multi-threaded).
- Expose the `HASH_LIST_SCAN` struct that `scan_manager.c` embeds inside `SCAN_ID.s` for hash-type scans.

## Public Entry Points

| Function | Phase | Description |
|---|---|---|
| `qexec_hash_join(thread_p, xasl, query_id, val_descr)` | open/all | Top-level entry; initialises manager, decides path, runs join |
| `hjoin_execute(thread_p, manager, context)` | execute | Run build+probe for one partition context |
| `hjoin_fetch_key(thread_p, fetch_info, ...)` | next | Read next tuple from a list file and extract join key |
| `qdata_build_hscan_key(thread_p, vd, regu_list, key)` | build/probe | Evaluate `regu_list` into `HASH_SCAN_KEY` using `val_descr` host variables |
| `qdata_hash_scan_key(key, ht_size, hash_method)` | build/probe | Hash a `HASH_SCAN_KEY`; for `HASH_METH_HASH_FILE` applies second-level `fhs_hash()` |
| `qdata_hscan_key_eq(key1, key2)` | probe | Equality predicate for MHT |
| `qdata_copy_hscan_key(...)` / `_without_alloc(...)` | probe | Deep-copy key, coercing types when probe and build domains differ |
| `fhs_create` / `fhs_destroy` | open / close | Create/destroy disk-backed extendible hash file |
| `fhs_insert(thread_p, fhsid, key, value_ptr)` | build | Insert `(key → TFTID)` into file hash |
| `fhs_search` / `fhs_search_next` | probe | Probe and iterate duplicate-key chains in file hash |

## Execution Path

```
qexec_hash_join()
  ├── hjoin_init_manager()            — populate HASHJOIN_MANAGER from XASL
  ├── hjoin_check_empty_inputs()      — short-circuit if either side empty
  ├── hjoin_try_partition()           — decides: SINGLE / PARTITION / PARALLEL
  │     ├── hjoin_check_partition()   — compare (sizeof(HENTRY_HLS) + sizeof(QFILE_TUPLE_SIMPLE_POS)) * min_tuples
  │     │                               against PRM_ID_MAX_HASH_LIST_SCAN_SIZE
  │     ├── hjoin_prepare_partition() — allocate partition list files
  │     └── hjoin_try_parallel()      — [SERVER_MODE only] try parallel_query pool
  │
  ├── [SINGLE]     hjoin_execute_internal()
  │     ├── hjoin_build()            — scan build-side list file, insert into MHT_HLS hash table
  │     └── hjoin_probe()            — scan probe-side list file, lookup each key, emit matches
  │
  ├── [PARTITION]  hjoin_execute_partitions()
  │     └── for each partition context: hjoin_execute() + hjoin_merge_qlist()
  │
  └── [PARALLEL]   parallel_query::hash_join::execute_partitions()   [SERVER_MODE]
```

## Hash Method Selection

`HASH_METHOD` in `query_hash_scan.h`:

| Value | Constant | Meaning |
|---|---|---|
| 0 | `HASH_METH_NOT_USE` | Hash join disabled |
| 1 | `HASH_METH_IN_MEM` | In-memory `MHT_HLS_TABLE` |
| 2 | `HASH_METH_HYBRID` | In-memory with overflow to file |
| 3 | `HASH_METH_HASH_FILE` | Disk-backed extendible hash (`FHSID`) |

The `HASH_LIST_SCAN` struct carries two union branches: `memory.hash_table` (`MHT_HLS_TABLE*`) and `file.hash_table` (`FHSID*`), selected by `hash_list_scan_type`.

> [!key-insight] Memory budget and spill decision
> `hjoin_check_partition` computes how many `(HENTRY_HLS + QFILE_TUPLE_SIMPLE_POS)` structs the smaller side would require. If that exceeds `PRM_ID_MAX_HASH_LIST_SCAN_SIZE` (system parameter), the join is split into `part_cnt = CEIL(estimated_bytes / (mem_limit * 0.8))` partitions, each small enough to fit in memory. The fill factor of 0.8 (`PARTITION_FILL_FACTOR`) prevents hash-table saturation. There is **no runtime spill mid-build**; partitioning is decided upfront based on tuple count estimates from `QFILE_LIST_ID.tuple_cnt`.

> [!key-insight] Serial vs parallel choice
> Parallelism is only considered **after** the partition decision confirms `part_cnt > 1`. `hjoin_try_parallel` (SERVER_MODE only, non-Windows) checks `xasl->parallelism` (set by the optimizer) against the `"parallel-query"` worker pool availability. On SA_MODE or Windows, the code asserts `px_worker_manager == NULL` and falls back to serial partitioned execution. This means the serial scan-hash module is always the fallback.

## Key Data Structures

### HASH_SCAN_KEY
```c
struct hash_scan_key {
  int    val_count;     // number of join columns
  bool   free_values;   // true → destructor frees DB_VALUE*
  DB_VALUE **values;    // array of pointers to join column values
};
```

### HASH_SCAN_VALUE (union)
```c
union hash_scan_value {
  void                    *data;  // for free()
  QFILE_TUPLE_SIMPLE_POS  *pos;   // temp-file tuple position (file mode)
  QFILE_TUPLE              tuple; // inline tuple data (memory mode)
};
```

### FHSID (File Hash Scan ID)
```c
struct file_hash_scan_id {
  EHID      ehid;          // directory file identifier
  VFID      bucket_file;   // bucket file
  unsigned short depth;    // global depth of the extendible hash directory
  char      alignment;     // slot alignment
};
```
`FHSID` is a disk-backed extendible hash: the directory page array doubles when the global depth increases; duplicate keys overflow into linked `dk_bucket` chains (`FHS_FLAG_INDIRECT = 0xFFFF`).

### TFTID (Temp File Tuple ID)
```c
struct temp_file_tuple_id {
  int   pageid;
  short volid;
  short offset;  // max 16K page → fits in short
};
```

## Constraints

- **Build mode**: `SERVER_MODE` or `SA_MODE` only (`#if defined(SERVER_MODE) || defined(SA_MODE)`). CS_MODE client does not execute hash builds.
- **Memory**: All `db_private_alloc` allocations tied to the `THREAD_ENTRY` private heap. Keys are freed via `qdata_free_hscan_key`; values via `qdata_free_hscan_value`; the MHT is freed by `hjoin_scan_clear` at context end.
- **Threading**: The `HASHJOIN_MANAGER.single_context` is always single-threaded. Parallel contexts share only `HASHJOIN_SHARED_JOIN_INFO` (protected by `std::mutex`) and per-partition list files.
- **Type coercion**: `HASHJOIN_DOMAIN_INFO.need_coerce_domains` is set when outer and inner join columns have different types or different precision/scale. Common domain is computed by `tp_more_general_type` + promotion rules; this is done once at manager init, not per-tuple.
- **Outer joins**: NULL-carrying tuples on the probe side are placed in the last partition; `hjoin_outer_fill_null_values` handles that partition specially to emit NULL-padded rows.
- **Error recovery on partition failure**: If `hjoin_build_partitions` fails (non-interrupt), the code clears the error and falls back to `HASHJOIN_STATUS_SINGLE`, re-trying the join in single-context mode.

## Lifecycle

```
open   : hjoin_init_manager() — alloc HASHJOIN_MANAGER, type_list, domain_info
start  : hjoin_check_partition() → choose path; hjoin_prepare_partition() if needed
next   : [per-tuple in probe loop]
         hjoin_fetch_key() → qdata_build_hscan_key() → qdata_hash_scan_key()
         → mht_get()/fhs_search() → hjoin_merge_tuple_to_list_id()
close  : hjoin_clear_manager() → hjoin_scan_clear() → fhs_destroy() or mht_destroy()
         → qfile_destroy_list() on all partition list IDs
```

Per-tuple probe cost: one `qfile_scan_list_next`, one `qdata_build_hscan_key` (evaluates regu variables), one `mht_get` or `fhs_search`, one `hjoin_merge_tuple` memcpy.

## Contrast with Parallel Hash Join

| Dimension | This module (serial) | [[components/parallel-hash-join|parallel-hash-join]] |
|---|---|---|
| Activation | Always the base; parallel is an upgrade path | Requires `SERVER_MODE`, non-Windows, `xasl->parallelism > 1`, worker pool available |
| Build phase | Single-threaded `hjoin_build()` | Workers each scan a partition concurrently via `parallel_query::hash_join::build_partitions` |
| Probe phase | Single-threaded `hjoin_probe()` | Workers each probe a partition concurrently |
| Shared state | None (single context) | `HASHJOIN_SHARED_JOIN_INFO.scan_mutex`, `stats_mutex`, `HASHJOIN_SHARED_SPLIT_INFO.part_mutexes` |
| Stats | Optional trace stats in `HASHJOIN_STATS` | Worker stats aggregated via `hjoin_trace_drain_worker_stats` |

## Related

- [[components/scan-manager|scan-manager]] — dispatcher that hands `S_LIST_SCAN` and `S_HASH_LIST_SCAN` to this module
- [[components/parallel-hash-join|parallel-hash-join]] — parallel upgrade path; shares `HASH_LIST_SCAN` and key/value helpers
- [[components/list-file|list-file]] — `QFILE_LIST_ID` / `qfile_open_list` / `qfile_scan_list_next`; both sides of the join are already spooled here before `qexec_hash_join` is called
- [[components/query-executor|query-executor]] — calls `qexec_hash_join` from `qexec_execute_mainblock`
- [[components/extendible-hash|extendible-hash]] — the `FHSID` extendible hash uses the same disk-resident structure as the general-purpose EH module
- [[Memory Management Conventions]] — `db_private_alloc` / `free_and_init` patterns
- [[Build Modes (SERVER SA CS)]] — `SERVER_MODE || SA_MODE` guard on all scan machinery
