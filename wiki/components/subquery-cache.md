---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/subquery_cache.c + src/query/subquery_cache.h"
status: active
purpose: "Per-execution result cache for correlated scalar subqueries; keyed by the correlated parameter DB_VALUE array; avoids re-executing the same subquery for duplicate outer-query rows"
key_files:
  - "src/query/subquery_cache.h (SQ_CACHE, SQ_KEY, SQ_VAL, sq_type enum, public API)"
  - "src/query/subquery_cache.c (sq_cache_initialize, sq_put, sq_get, sq_make_key, sq_free_key)"
  - "src/query/xasl.h (XASL_USES_SQ_CACHE flag; xasl_node::sq_cache pointer)"
build_mode: "SERVER_MODE | SA_MODE"
public_api:
  - "sq_cache_initialize(xasl) → int   — allocate MHT_TABLE for this subquery node"
  - "sq_put(thread_p, key, xasl, regu_var) → int   — store result; returns ER_FAILED if over size limit"
  - "sq_get(thread_p, key, xasl, regu_var) → bool   — lookup; fills regu_var on hit"
  - "sq_make_key(thread_p, xasl) → SQ_KEY *   — snapshot current correlated parameter values"
  - "sq_free_key(thread_p, key) → void"
  - "sq_cache_destroy(thread_p, sq_cache) → void   — called at XASL end-of-execution"
tags:
  - component
  - cubrid
  - xasl
  - subquery
  - cache
  - query
related:
  - "[[components/xasl|xasl]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/memoize|memoize]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# Subquery Cache — Correlated Scalar Subquery Result Cache

The subquery cache (`sq_*`) is a per-execution-run, per-subquery result memo table. It is attached to an `XASL_NODE` that represents a **correlated scalar subquery** — a subquery whose result depends on columns from the outer query's current row. Without a cache, such a subquery would execute once per outer-query tuple, even when many outer tuples produce identical values for the correlated columns.

> [!key-insight] Distinct from [[components/memoize|memoize]] and [[components/xasl-cache|xasl-cache]]
> There are three different caching layers in CUBRID's query execution:
>
> | Cache | Granularity | Lifetime | Key |
> |-------|-------------|----------|-----|
> | `xcache` (XASL cache) | Compiled query plan (stream) | Cross-session, persists across executions | SHA-1 of rewritten SQL text |
> | `subquery_cache` (this) | Scalar subquery result value | Per-statement execution, one per subquery XASL node | Current values of correlated outer-query columns |
> | `memoize` ([[components/memoize|memoize]]) | Uncorrelated subquery result | Per-statement; result computed once and reused | No key needed (uncorrelated = no outer dependency) |
>
> `subquery_cache` specifically handles **correlated** subqueries — the memoize path handles uncorrelated ones (flag `XASL_ZERO_CORR_LEVEL`).

---

## Data Structures

```c
struct sq_cache {
  SQ_KEY    *sq_key_struct;  // template key (holds pointers into outer scan value slots)
  MHT_TABLE *ht;             // hash table: SQ_KEY → SQ_VAL
  UINT64     size_max;       // PRM_ID_MAX_SUBQUERY_CACHE_SIZE
  UINT64     size;           // current estimated memory usage
  bool       enabled;
  struct { int hit; int miss; } stats;
};

struct sq_key {
  DB_VALUE **dbv_array;    // snapshot of correlated parameter DB_VALUEs
  int        n_elements;
};

struct sq_val {
  SQ_REGU_VALUE val;       // either dbvalptr (TYPE_CONSTANT) or exists bool (TYPE_LIST_ID)
  REGU_DATATYPE type;
};
```

`sq_cache` is embedded in the subquery `XASL_NODE` via `xasl->sq_cache`. The `XASL_USES_SQ_CACHE` flag (bit 18 of `XASL_NODE.flag`) marks nodes eligible for caching.

---

## Key Design

The cache key is a snapshot of the correlated parameter values at the time the outer query processes a given row. `sq_make_key` copies the current `DB_VALUE *` values from `SQ_CACHE_KEY_STRUCT(xasl)->dbv_array` (which points into the outer scan's value list).

Hash function: `ROTL32`-based rolling XOR over `mht_get_hash_number()` of each key element. Comparison: element-by-element `mht_compare_dbvalues_are_equal`.

---

## Lifecycle

```
1. XASL generation (client, !SERVER_MODE):
   ├─ Correlated subquery XASL node receives XASL_USES_SQ_CACHE flag
   └─ sq_cache pointer left NULL until first use

2. First execution of outer query row (server):
   ├─ qexec_execute_mainblock sees XASL_USES_SQ_CACHE flag
   ├─ sq_make_key() — snapshot correlated param values
   ├─ sq_get() → miss (cache not yet populated)
   ├─ Execute subquery normally
   └─ sq_put() — store key → result in MHT

3. Subsequent outer rows with same correlated values:
   ├─ sq_make_key() — snapshot param values
   ├─ sq_get() → hit — sq_unpack_val() writes result into regu_var
   └─ subquery execution skipped entirely

4. End of statement execution:
   └─ sq_cache_destroy() — mht_destroy + free all keys/values
```

---

## Memory and Adaptive Disabling

The cache uses `PRM_ID_MAX_SUBQUERY_CACHE_SIZE` as a hard memory ceiling. `sq_put()` estimates the per-entry size (key values + value + hash overhead) and silently disables the cache (`SQ_CACHE_ENABLED = false`) when the next entry would exceed the limit.

An adaptive quality check runs in `sq_get()` when the cache is >60% full:

```c
if (SQ_CACHE_SIZE(xasl) > SQ_CACHE_SIZE_MAX(xasl) * 0.6) {
  if (SQ_CACHE_HIT(xasl) / SQ_CACHE_MISS(xasl) < SQ_CACHE_MIN_HIT_RATIO) {
    SQ_CACHE_ENABLED(xasl) = false;  // evict self
    return false;
  }
}
```

`SQ_CACHE_MIN_HIT_RATIO = 9` means the hit:miss ratio must be at least 9:1 (90% hit rate) once the cache is substantially full. If below this threshold, caching is abandoned for the remainder of the statement execution.

> [!warning] Cache is per-execution, not per-plan
> `sq_cache` is initialized lazily on first lookup and destroyed at the end of each execution. Unlike the XASL cache, no results survive across statement boundaries. A second execution of the same prepared statement starts with an empty subquery cache.

---

## Constraints

| Constraint | Detail |
|-----------|--------|
| Build mode | `SERVER_MODE | SA_MODE` (hash table calls are gated) |
| Memory ceiling | `PRM_ID_MAX_SUBQUERY_CACHE_SIZE` (default 2 MB based on SQ_CACHE_EXPECTED_ENTRY_SIZE=512 → ~4096 entries) |
| Value types | Only `TYPE_CONSTANT` (scalar) and `TYPE_LIST_ID` (EXISTS result) are stored; other REGU types bypassed |
| Correlated only | Uncorrelated subqueries (`XASL_ZERO_CORR_LEVEL`) use the memoize path instead |
| Thread safety | Per-XASL, single-threaded execution; no locking needed |
| Adaptive | Disables itself when hit rate < 90% at >60% capacity — prevents wasted memory on cardinality-diverse outer tables |

---

## Related

- [[components/xasl|xasl]] — `XASL_NODE.sq_cache` field; `XASL_USES_SQ_CACHE` flag
- [[components/regu-variable|regu-variable]] — `REGU_VARIABLE_CORRELATED` flag marks the outer column references feeding the key
- [[components/query-executor|query-executor]] — calls `qexec_execute_subquery_for_result_cache` when `XASL_USES_SQ_CACHE` is set
- [[components/memoize|memoize]] — the parallel mechanism for uncorrelated subqueries (no key needed)
- [[components/xasl-cache|xasl-cache]] — the server-side plan cache (completely different granularity)
- [[Query Processing Pipeline]]
- Source: [[sources/cubrid-src-query|cubrid-src-query]]
