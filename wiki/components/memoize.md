---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/memoize.hpp"
status: active
purpose: "In-query memoization of repeated subquery evaluations: cache DB_VALUE key→value mappings in a fixed-size in-memory hash map with hit-ratio self-disable"
key_files:
  - "memoize.hpp (memoize::storage class definition and C extern API)"
  - "memoize.cpp (implementation)"
public_api:
  - "new_memoize_storage(thread_p, xasl) → int  (C extern)"
  - "clear_memoize_storage(thread_p, xasl)       (C extern)"
  - "memoize_get(thread_p, xasl, *success, *is_ended) → int  (C extern)"
  - "memoize_put(thread_p, xasl, *success) → int  (C extern)"
  - "memoize_put_nullptr(thread_p, xasl, *success) → int  (C extern)"
tags:
  - component
  - cubrid
  - query
  - memoize
  - cache
  - subquery
related:
  - "[[components/query|query]]"
  - "[[components/query-executor|query-executor]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# `memoize.hpp` — Subquery Memoization

A per-execution in-memory memoization table for correlated or repeated subquery evaluation. When the same input key (set of `DB_VALUE`s) is seen multiple times during a single query, the memoize cache returns the previously computed output instead of re-executing the subquery XASL node.

## Design

### `memoize::storage`

```cpp
class storage {
  const size_t m_max_storage_size;  // max bytes for this cache
  const int m_key_cnt;              // number of key DB_VALUEs
  const int m_value_cnt;            // number of output DB_VALUEs

  // key: vector<DB_VALUE> keyed by pointer with custom hash+equal
  std::unordered_multimap<key*, value*,
                          key::hash, key::equal,
                          fixed_allocator<...>> m_key_value_map;

  bool disabled;           // self-disabled if hit ratio too low
  bool has_range;
  bool key_changed;
  bool current_key_joined;

  size_t hit;   // hit counter
  size_t miss;  // miss counter
  struct timeval m_elapsed_time;  // for profiling
};
```

### Key and Value

```cpp
struct key   { std::vector<DB_VALUE> m_values; size_t m_size; };
struct value { std::vector<DB_VALUE> m_values; size_t m_size; };
```

Both use `DB_VALUE` vectors. The custom `key::hash` and `key::equal` functors handle CUBRID type-aware hashing and comparison.

### Fixed-Size Allocator

`fixed_allocator<T, false>` (`cubmem::fixed_size_alloc::allocator`) is used for hash map nodes to avoid `malloc/free` overhead per entry. Memory is pre-allocated up to `m_max_storage_size`.

### C Extern Wrapper

The class is wrapped in `extern "C"` functions for use from `query_executor.c` (compiled as C++17 but calls C-linkage interfaces):

```c
int  new_memoize_storage(THREAD_ENTRY*, xasl_node*);
void clear_memoize_storage(THREAD_ENTRY*, xasl_node*);
int  memoize_get(THREAD_ENTRY*, xasl_node*, bool *success, bool *is_ended);
int  memoize_put(THREAD_ENTRY*, xasl_node*, bool *success);
int  memoize_put_nullptr(THREAD_ENTRY*, xasl_node*, bool *success);
```

`memoize_put_nullptr` stores a NULL result (e.g., correlated subquery returns no rows — the NULL result is itself cacheable).

## Self-Disable Mechanism

> [!key-insight] Hit-ratio self-disable
> After `MEMOIZE_FREE_ITERATION_LIMIT = 1000` insertions, if `hit / (hit + miss) < MEMOIZE_HIT_RATIO_THRESHOLD = 0.5`, the storage marks itself `disabled = true`. Subsequent `memoize_get` / `memoize_put` calls are no-ops. This prevents the cache from consuming memory and time when the key distribution is highly unique (e.g., every row has a different subquery input).

## Lifecycle

```
new_memoize_storage()    ← called when XASL node is initialized
  ↓
[per tuple evaluation:]
memoize_get()            ← check cache; *success=true if hit, *is_ended=true if NULL result cached
  if miss:
    evaluate subquery
    memoize_put() / memoize_put_nullptr()
  ↓
clear_memoize_storage()  ← called at qexec_clear_xasl time
```

## Memory Tracking

`storage` tracks three memory components for merge statistics: `m_key_sz`, `m_value_sz`, `m_hash_sz`. `get_size_for_merge_stats()` returns these as a `std::vector<size_t>` for aggregation across parallel worker clones.

## Related

- Parent: [[components/query|query]]
- [[components/query-executor|query-executor]] — `memoize_get` / `memoize_put` called inside `qexec_execute_mainblock` for subquery nodes
- [[Query Processing Pipeline]] — optimization layer: avoids repeated subquery evaluation for repeated input keys
