---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/filter_pred_cache.c"
status: active
purpose: "Cache for compiled filter predicate expressions (pred_expr_with_context) keyed by BTID, to avoid repeated deserialization for filtered index scans"
key_files:
  - "filter_pred_cache.c (fpcache_* implementation)"
  - "filter_pred_cache.h (public API)"
public_api:
  - "fpcache_initialize(thread_p) → int"
  - "fpcache_finalize(thread_p)"
  - "fpcache_claim(thread_p, btid, or_pred, pred_expr) → int"
  - "fpcache_retire(thread_p, class_oid, btid, filter_pred) → int"
  - "fpcache_remove_by_class(thread_p, class_oid)"
  - "fpcache_drop_all(thread_p)"
  - "fpcache_dump(thread_p, fp)"
tags:
  - component
  - cubrid
  - query
  - cache
  - predicate
  - index
related:
  - "[[components/query|query]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/query-executor|query-executor]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# `filter_pred_cache.c` — Filter Predicate Cache

The filter predicate cache (fpcache) stores compiled `pred_expr_with_context` objects for **filtered indexes** (partial/conditional indexes). Without the cache, every index scan that uses a filter predicate would need to deserialize the predicate expression from disk on each execution.

## Problem Being Solved

A filtered index in CUBRID has a filter predicate stored in the index's metadata. When the index scan opens (`scan_open_index_scan`), the predicate must be deserialized from its `or_predicate` (on-disk representation) into an executable `pred_expr_with_context`. This deserialization is not free, and for repeated queries (e.g., tight loops or high-frequency OLTP) it would happen on every execution.

The fpcache amortizes this cost by caching the compiled `pred_expr_with_context` per `BTID`.

## API Pattern: Claim / Retire

```c
// Try to get a cached predicate for this index:
fpcache_claim(thread_p, btid, or_pred, &pred_expr)
  // returns existing entry if available, or deserializes or_pred into pred_expr

// Return the predicate back to the cache when done:
fpcache_retire(thread_p, class_oid, btid, filter_pred)
```

This is a **lend/return** pattern (not a read-only cache): a claimed entry is removed from the cache and exclusively held by the scan. On retire, it is returned. This avoids thread-safety issues around mutable predicate state.

> [!key-insight] Claim = exclusive lease
> Unlike a typical read cache, `fpcache_claim` removes the entry from the cache and gives exclusive ownership to the caller. Concurrent scans on the same index each get their own deserialized copy (or the cache supplies one and deserializes a fresh one for the next caller). This avoids locking the predicate during scan evaluation.

## Invalidation

- `fpcache_remove_by_class(class_oid)` — invalidate all filter predicates for a given class (called on DDL: `DROP INDEX`, `ALTER TABLE`).
- `fpcache_drop_all()` — global flush (called during shutdown or emergency).

## Cache Key

The cache is keyed by `BTID` (B-tree identifier). Each entry carries:
- The compiled `pred_expr_with_context` (executable predicate tree with `HEAP_CACHE_ATTRINFO` inside)
- The associated `class_oid` for invalidation by class

## Diagnostics

`fpcache_dump(thread_p, fp)` prints cache statistics and current entries to `fp`. Useful for debugging partial index scan performance.

## Related

- Parent: [[components/query|query]]
- [[components/scan-manager|scan-manager]] — calls `fpcache_claim` during `scan_open_index_scan` for filtered indexes
- [[components/query-executor|query-executor]] — calls `qexec_clear_pred_context` to release predicate contexts
