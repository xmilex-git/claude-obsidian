---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_heap_scan/px_heap_scan_join_info.{hpp,cpp}"
status: active
purpose: "Shared scan-state container for parallel heap scan: per-class metadata + per-block status tracking"
key_files:
  - "px_heap_scan_join_info.hpp (scan_info struct, STL containers)"
  - "px_heap_scan_join_info.cpp (methods + mutex protection for status mutations)"
public_api:
  - "parallel_heap_scan::scan_info struct — per-class scan record"
tags:
  - component
  - cubrid
  - parallel
  - query
  - heap-scan
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]]"
  - "[[components/parallel-heap-scan-task|parallel-heap-scan-task]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/heap-file|heap-file]]"
created: 2026-04-23
updated: 2026-04-23
---

# `px_heap_scan_join_info` — Per-Class Scan Info

Holds the per-class state a parallel heap-scan worker needs to perform (and report) its scan. Used by `parallel_heap_scan::manager<T>` to track which class-blocks have been qualified, consumed, and completed.

## `scan_info` struct

```cpp
struct scan_info
{
  /* read-only section (set once, no mutex needed) */
  OID oid;                      // class OID
  HFID hfid;                    // class HFID
  BTID btid;                    // index id (INVALID when full heap scan)
  QFILE_LIST_ID *list_id;       // output list file identifier
  TARGET_TYPE target_type;      // TARGET_CLASS / TARGET_LIST / ...
  ACCESS_METHOD access_method;  // SEQ_SCAN / INDEX_SCAN

  /* writable section (mutex-protected) */
  SCAN_STATUS status;           // START / BLOCK / END / QUALIFIED / ...
  bool qualified_block;         // current block matched the predicate
};
```

Read-only fields are populated once during scan setup and never modified, so concurrent readers need no synchronization on them. Only `status` and `qualified_block` mutate — the owning container (`join_context` / `manager`) guards those with a mutex.

## Usage pattern

```
scan_info (read-only metadata)
    │
    │  looked up by class OID
    ▼
parallel_heap_scan::manager<T>::on_scan_start
    │
    │  workers mutate status under mutex
    ▼
parallel_heap_scan::manager<T>::on_scan_complete
```

A `std::map<OID, scan_info>` keyed by class OID is the usual container. Partition-aware scans hold multiple entries (one per partition heap).

> [!key-insight] Split RW sections in one struct
> CUBRID chose to keep read-only metadata and writable status in the SAME struct (rather than two structs). The header comment documents the split to prevent future readers from assuming everything is mutex-protected. Adding another mutable field requires re-reading the discipline boundary.

## Lifecycle

1. **Setup (main thread)**: `parallel_heap_scan::manager` creates one `scan_info` per class/partition; fills the read-only section.
2. **Dispatch**: workers receive references; each worker probes `status` under mutex before processing a block.
3. **Scan progress**: workers atomically transition `status` (`START → BLOCK → ... → END`), set `qualified_block` if a block produced matching rows.
4. **Teardown**: when all workers report `END`, the main thread aggregates — list file contents belong to the worker who wrote them, but the scan_info tells the main thread where to find them.

## Constraints

- **Threading**: mutex required for any mutation of `status` / `qualified_block`. Readers of RO fields are lock-free.
- **Build-mode guard**: `#if !defined (SERVER_MODE) && !defined (SA_MODE)` via parent module. Server-only.
- **Memory**: struct is trivially-copyable; holds a non-owning pointer to `QFILE_LIST_ID` (owned by the list-file layer).

## Related

- Parent hub: [[components/parallel-heap-scan|parallel-heap-scan]]
- Consumers: [[components/parallel-heap-scan-task|task]], [[components/parallel-heap-scan-input-handler|input-handler]], [[components/parallel-heap-scan-result-handler|result-handler]]
- Scan primitives: [[components/scan-manager|scan-manager]] (`SCAN_STATUS` enum), [[components/heap-file|heap-file]] (OID/HFID)
