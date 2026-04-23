---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/xasl_cache.c + src/query/xasl_cache.h"
status: active
purpose: "Server-side plan cache keyed by SHA-1 of the rewritten query text; stores the packed XASL stream and optionally pre-deserialized XASL_NODE clones; shared across sessions; evicted by memory pressure (LRU + binary-heap candidate selection) or by time threshold"
key_files:
  - "src/query/xasl_cache.h (XASL_CACHE_ENTRY, XASL_CLONE, XCACHE_RELATED_OBJECT, EXECUTION_INFO, xcache_find_sha1, xcache_insert, xcache_unfix)"
  - "src/query/xasl_cache.c (global xcache singleton, latch-free hashmap, xcache_cleanup binary-heap eviction)"
build_mode: "SERVER_MODE | SA_MODE only (#error guard in xasl_cache.h)"
public_api:
  - "xcache_initialize(thread_p) → int   — called once at server boot"
  - "xcache_finalize(thread_p) → void"
  - "xcache_find_sha1(thread_p, sha1, search_mode, &entry, &rt_check) → int   — lookup by SHA-1"
  - "xcache_find_xasl_id_for_execute(thread_p, xid, &entry, xclone) → int   — lookup + lock + clone"
  - "xcache_insert(thread_p, context, stream, n_oid, class_oids, class_locks, tcards, &entry) → int"
  - "xcache_unfix(thread_p, entry) → void   — decrement ref-count; erase from hash if last"
  - "xcache_retire_clone(thread_p, entry, xclone) → void   — return clone to entry's pool"
  - "xcache_remove_by_oid(thread_p, oid) → void   — invalidate all plans touching OID"
  - "xcache_remove_by_sha1(thread_p, sha1) → void"
  - "xcache_drop_all(thread_p) → void   — full flush"
  - "xcache_dump(thread_p, fp) → void   — diagnostics"
  - "xcache_invalidate_qcaches(thread_p, oid) → int   — invalidate list-file caches for OID"
tags:
  - component
  - cubrid
  - xasl
  - cache
  - query
related:
  - "[[components/xasl|xasl]]"
  - "[[components/xasl-stream|xasl-stream]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/session|session]]"
  - "[[components/list-file|list-file]]"
  - "[[components/mvcc|mvcc]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# XASL Cache — Server-Side Plan Cache

The XASL cache (`xcache_*`) is the server-side repository for compiled query plans. After the client serialises an `XASL_NODE` tree to a byte stream, it sends the stream to the server. On the first execution of any query text, the server stores that stream in the XASL cache keyed by the SHA-1 hash of the rewritten ("hash") query text. Subsequent executions from any session look up the cache by SHA-1 and skip the expensive client-side parse + optimize + serialize round-trip.

> [!key-insight] SHA-1 is the primary key, time_stored is the versioning key
> `xcache_find_sha1` matches by SHA-1 only. `xcache_find_xasl_id_for_execute` additionally validates `time_stored` (sec + usec). A session holding a stale `XASL_ID` (from a previous prepare cycle) will fail the `time_stored` check and transparently trigger a recompile. This is the primary "stale reference" safety mechanism — sessions do not crash, they just pay one extra round-trip.

---

## Entry Structure

```c
struct xasl_cache_ent {
  XASL_ID           xasl_id;        // SHA-1 + time_stored + cache_flag (ref-count + status bits)
  XASL_STREAM       stream;         // packed XASL buffer (char * + size); the source of truth
  EXECUTION_INFO    sql_info;       // sql_hash_text (hash key), sql_user_text, sql_plan_text
  XCACHE_RELATED_OBJECT *related_objects; // classes/serials touched by this plan + their locks + tcard
  int               n_related_objects;
  struct timeval    time_last_used;
  INT64             ref_count;      // lifetime hit counter
  INT64             clr_count;      // qfile list-cache clear count
  int               list_ht_no;     // result-cache list hash table index (-1 if none)
  XASL_CLONE       *cache_clones;   // pre-deserialized XASL_NODE clones (pool)
  XASL_CLONE        one_clone;      // embedded fast-path clone slot (avoids malloc for 1-clone case)
  int               n_cache_clones;
  int               cache_clones_capacity;
  pthread_mutex_t   cache_clones_mutex;
  INT64             time_last_rt_check; // last recompile-threshold check time
};
```

### `XASL_CLONE`

```c
struct xasl_clone {
  xasl_unpack_info *xasl_buf;  // unpack arena (owns all sub-allocations)
  XASL_NODE        *xasl;      // deserialized plan tree
};
```

Clones are pre-unpacked XASL trees ready for immediate re-execution, allowing concurrent sessions to skip the `stx_map_stream_to_xasl` deserialization step.

---

## Cache Lifecycle

```
xcache_insert()
  ├─ SHA-1 set from compile_context.sha1
  ├─ time_stored = gettimeofday()
  ├─ xcache_Hashmap.find_or_insert()
  │    ├─ hit: return existing entry (already fixed)
  │    └─ miss: allocate XASL_CACHE_ENTRY, copy stream + sql_info + related_objects
  └─ returns fixed entry (cache_flag fix-count incremented)

xcache_find_sha1() / xcache_find_xasl_id_for_execute()
  ├─ Hashmap.find(sha1 key)
  │    └─ xcache_compare_key() atomically CAS increments cache_flag fix-count on match
  ├─ Lock all related_objects (class locks acquired via lock_object)
  │    └─ Validates plan is still schema-consistent
  ├─ If xcache_uses_clones():
  │    ├─ pthread_mutex_lock(cache_clones_mutex)
  │    ├─ pop clone from cache_clones[] stack
  │    └─ if empty: fall through to stx_map_stream_to_xasl()
  └─ returns (entry, clone) pair to caller

xcache_unfix()
  ├─ ATOMIC_TAS time_last_used
  ├─ ATOMIC_INC ref_count
  ├─ CAS decrement fix-count in cache_flag
  └─ if last fixer + XCACHE_ENTRY_MARK_DELETED:
       ├─ xcache_clone_decache() all remaining clones
       ├─ qfile_clear_list_cache() if list_ht_no >= 0
       └─ xcache_Hashmap.erase()

xcache_retire_clone()
  ├─ if n_cache_clones < xcache_Max_clones && entry_size < xcache_Max_plan_size:
  │    push clone back to cache_clones[] stack   ← reuse
  └─ else: xcache_clone_decache() (free unpack arena)
```

---

## Eviction Policy

There is **no true LRU eviction**. Eviction is triggered lazily at insert time when `xcache_need_cleanup()` detects one of two conditions:

| Trigger | Condition | Strategy |
|---------|-----------|----------|
| `XCACHE_CLEANUP_FULL_MEMORY` | `memory_usage_cache + memory_usage_clone > xcache_Soft_limit` | Binary-heap LRU: collect candidates by scanning hash; sort into min-heap by `time_last_used`; evict the least recently used until `XCACHE_CLEANUP_RATIO × soft_limit` bytes freed |
| `XCACHE_CLEANUP_TIMEOUT` | `now - last_cleaned_time > xcache_Time_threshold` (default 360 s = 6 min) | Time sweep: remove all entries with `time_last_used` older than threshold |

Cleanup is guarded by an atomic CAS on `xcache_Cleanup_flag`; only one thread performs cleanup at a time.

### Memory limits

| Variable | Formula | Meaning |
|----------|---------|---------|
| `xcache_Soft_capacity` | `PRM_ID_XASL_CACHE_MAX_ENTRIES` | Entry count hint |
| `xcache_Hard_limit` | `soft_capacity × 1024 × 128` bytes | Absolute memory ceiling |
| `xcache_Soft_limit` | `hard_limit × 0.8` | Cleanup trigger threshold |
| `xcache_Max_plan_size` | `(hard_limit - soft_limit) / UNPACK_SCALE` | Max XASL stream size for a single plan to be clone-cached |

> [!key-insight] Eviction is not triggered continuously — it piggybacks on insert
> `xcache_cleanup()` is called inside `xcache_insert()` only after the new entry is already in the hash. This means total memory can briefly exceed `soft_limit` before cleanup runs. `hard_limit` is not enforced by the allocator — it is informational metadata only.

---

## Concurrency Model

The hashmap is a **latch-free** `cubthread::lockfree_hashmap` (template over SHA-1 key, `XASL_CACHE_ENTRY` value). Ref-counting uses 24-bit fix-count packed into `xasl_id.cache_flag` (bits 0–23) alongside status flags in bits 24–31:

| Flag bit | Constant | Meaning |
|----------|----------|---------|
| 0x80000000 | `XCACHE_ENTRY_MARK_DELETED` | Entry is being deleted |
| 0x40000000 | `XCACHE_ENTRY_TO_BE_RECOMPILED` | Recompile in progress |
| 0x20000000 | `XCACHE_ENTRY_WAS_RECOMPILED` | Old plan, pending deletion |
| 0x10000000 | `XCACHE_ENTRY_SKIP_TO_BE_RECOMPILED` | Inserter skips old plan |
| 0x08000000 | `XCACHE_ENTRY_CLEANUP` | Under cleanup eviction |
| 0x04000000 | `XCACHE_ENTRY_RECOMPILED_REQUESTED` | Recompile requested, client notified |
| 0x00FFFFFF | `XCACHE_ENTRY_FIX_COUNT_MASK` | Reader count |

Multiple sessions can simultaneously hold a fix on the same entry — sharing is explicit by design. Clone access (push/pop from `cache_clones[]`) is guarded by a per-entry `pthread_mutex_t cache_clones_mutex`; the main hashmap operations are mutex-free.

---

## Recompile Flow

> [!key-insight] Stale plan detection uses runtime statistics, not schema version
> The recompile threshold (`xcache_check_recompilation_threshold`) fires when a table's heap cardinality has changed by >20% (or 2× for small tables) since the plan was compiled, AND at least `XCACHE_RT_TIMEDIFF_IN_SEC` (360 s) has elapsed since the last check. The flow is a 5-step client-server negotiation:
>
> 1. Execute hits cache, RT check fires → server returns `ER_QPROC_XASLNODE_RECOMPILE_REQUESTED`
> 2. Client (CAS) re-sends `prepare_query` without XASL
> 3. Server returns existing plan + `XASL_CACHE_RECOMPILE_PREPARE` flag
> 4. Client regenerates XASL, sends second `prepare_query` with new stream
> 5. Server inserts new entry (old entry transitions `WAS_RECOMPILED → MARK_DELETED`)

---

## Session Interaction

Sessions communicate with the XASL cache through `XASL_ID` tokens stored in `LOG_TDES.xasl_id`. The session layer ([[components/session|session]]) holds a prepared-statement slot that stores only the `XASL_ID` (SHA-1 + time_stored); it does not keep a pointer to the `XASL_CACHE_ENTRY`. Every re-execution calls `xcache_find_xasl_id_for_execute`, which:
1. Looks up by SHA-1.
2. Validates `time_stored` matches (schema change / recompile check).
3. Acquires class locks for schema-consistent execution.
4. Pops or creates a clone.

> [!warning] Prepared statement token becomes stale after schema change
> If a `DROP TABLE` or `ALTER TABLE` invalidates the plan, `xcache_remove_by_oid` marks the entry `MARK_DELETED`. The session's stored `XASL_ID` will fail the `time_stored` check on next execution, transparently triggering re-prepare without an error visible to the application.

---

## Constraints

| Constraint | Detail |
|-----------|--------|
| Build mode | `SERVER_MODE` or `SA_MODE` only; `#error` guard prevents CS_MODE include |
| Thread safety | Latch-free hashmap for lookup/insert/delete; per-entry mutex for clone pool |
| Memory | Hard limit not enforced by allocator — only by cleanup policy |
| Rate limiting | No rate limiting on vacuum/cleanup; one cleanup thread at a time |
| Clone size | Clones only cached if plan stream < `xcache_Max_plan_size` |
| Disabled path | `PRM_ID_XASL_CACHE_MAX_ENTRIES <= 0` → `xcache_Enabled = false` → all lookups/inserts are no-ops |

---

## Related

- [[components/xasl|xasl]] — `XASL_NODE` structure and the stream format stored in each entry
- [[components/xasl-stream|xasl-stream]] — `stx_map_stream_to_xasl` is called on cache miss to create a clone
- [[components/query-executor|query-executor]] — `qexec_execute_mainblock` drives the XASL plan obtained from cache
- [[components/session|session]] — prepared statement cache stores XASL_IDs; cache entry pointer is NOT stored
- [[components/list-file|list-file]] — `list_ht_no` links to a result-cache hash table for parameter-bound result caching
- [[Build Modes (SERVER SA CS)]]
- [[Query Processing Pipeline]]
- Source: [[sources/cubrid-src-query|cubrid-src-query]]
