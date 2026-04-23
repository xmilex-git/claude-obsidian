---
type: component
parent_module: "[[components/base|base]]"
path: "src/base/"
status: developing
purpose: "Lock-free hash maps, freelists, circular queues, and bitmaps using epoch-based reclamation — two API generations (modern C++ and legacy C)"
key_files:
  - "lockfree_hashmap.hpp/cpp (modern: lockfree::hashmap<Key,T>)"
  - "lockfree_freelist.hpp (modern: lockfree::freelist<T>)"
  - "lockfree_transaction_system.hpp/cpp (lockfree::tran::system — index management)"
  - "lockfree_transaction_table.hpp/cpp (per-structure tx table)"
  - "lockfree_transaction_descriptor.hpp/cpp (per-thread: active tx id + retired node list)"
  - "lockfree_transaction_reclaimable.hpp (base class for nodes that can be reclaimed)"
  - "lockfree_circular_queue.hpp (MPMC fixed-capacity power-of-2 queue)"
  - "lockfree_bitmap.hpp/cpp (CAS-based index allocation)"
  - "lockfree_address_marker.hpp (pointer low-bit marking for logical deletion)"
  - "lock_free.c/h (legacy: LF_HASH_TABLE, LF_FREELIST, LF_TRAN_SYSTEM)"
tags:
  - component
  - cubrid
  - lock-free
  - concurrent
related:
  - "[[components/base|base]]"
  - "[[components/parallel-task-queue|parallel-task-queue]]"
  - "[[components/parallel-query|parallel-query]]"
  - "[[components/memory-alloc|memory-alloc]]"
created: 2026-04-23
updated: 2026-04-23
---

# `lockfree` — Lock-Free Data Structures

CUBRID's lock-free infrastructure lives entirely in `src/base/`. It provides two API generations — a modern C++17 template layer and a legacy C layer — sharing the same epoch-based reclamation concept.

## Two Generations

| Aspect | Modern (`lockfree::` namespace) | Legacy (`lock_free.c`) |
|--------|---------------------------------|------------------------|
| Language | C++17 templates | C |
| Hash map | `lockfree::hashmap<Key, T>` | `LF_HASH_TABLE` |
| Freelist | `lockfree::freelist<T>` | `LF_FREELIST` |
| Tx system | `lockfree::tran::system` | `LF_TRAN_SYSTEM` |
| Tx entry | `lockfree::tran::descriptor` | `LF_TRAN_ENTRY` |
| Reclamation | Epoch-based (same algorithm) | Epoch-based |
| Status | Preferred for new code | Maintained, not extended |

## Modern Layer Architecture

```
lockfree::hashmap<Key, T>
        │
        ├── lockfree::freelist<T>          node recycling pool
        │
        └── lockfree::tran::table          per-structure transaction table
                │
                ├── lockfree::tran::descriptor   per-thread (active tx id + retired list)
                │
                └── lockfree::tran::system       index manager
                        │
                        └── lockfree::bitmap     CAS-based slot allocation
```

## `lockfree::hashmap<Key, T>`

```cpp
template <class Key, class T>
class hashmap {
  void init(tran::system &transys, size_t hash_size,
            size_t freelist_block_size, size_t freelist_block_count,
            lf_entry_descriptor &edesc);
  void destroy();

  T *find(tran::index tran_index, Key &key);
  bool find_or_insert(tran::index tran_index, Key &key, T *&entry);
  bool insert(tran::index tran_index, Key &key, T *&entry);
  bool erase(tran::index tran_index, Key &key);
  bool erase_locked(tran::index tran_index, Key &key, T *&locked_entry);
  void unlock(tran::index tran_index, T *&entry);

  T *freelist_claim(tran::index tran_index);   // get recycled node
  void freelist_retire(tran::index tran_index, T *&entry);  // return to pool

  void start_tran(tran::index tran_index);
  void end_tran(tran::index tran_index);
};
```

> [!key-insight] Logical deletion via address marking
> `lockfree_address_marker.hpp` uses the low bit of a pointer to mark a node as logically deleted. A marked pointer is invisible to traversals but still referenced — the node is only physically reclaimed once all concurrent readers have exited.

## Epoch-Based Reclamation (`lockfree::tran`)

The transaction system solves the ABA problem for CAS-based lock-free structures. Nodes are never reclaimed immediately — they enter a "retired" state and wait until all concurrent readers have advanced.

**Algorithm (per retire):**
1. Node is unlinked from the structure (logically deleted).
2. `freelist_retire(tran_index, node)` — node appended to the descriptor's retired list; global transaction ID is incremented.
3. At `end_tran()` — the descriptor's retired list is scanned; nodes whose retirement ID precedes all currently-active transactions' start IDs are safe to reclaim.

```
Thread A (active, tx_id=5)   Thread B (retiring node N, retirement_id=5)
     │                                │
     │ start_tran() → saves id=5      │ retire(N) → marks N, increments global to 6
     │                                │
     │ reading N ...                  │ end_tran() → wants to reclaim N
     │                                │   sees Thread A still at tx_id=5 ≤ retirement 5
     │                                │   → defers reclaim, N stays in retired list
     │ end_tran()                     │
                                      │ next end_tran() → Thread A is gone → reclaim N
```

## `lockfree::freelist<T>`

```cpp
template <class T>
class freelist {
  void init(tran::system &transys, size_t block_size, size_t block_count,
            lf_entry_descriptor &edesc);
  T *claim(tran::index tran_index);
  void retire(tran::index tran_index, T *&entry);
};
```

Maintains a pool of pre-allocated `T` nodes. `claim()` pops from the pool (or allocates a new block). `retire()` pushes back (deferred reclaim via transaction system).

## `lockfree::circular_queue<T>` (MPMC)

```cpp
// Fixed-capacity, power-of-2 size, multi-producer multi-consumer
lockfree_circular_queue.hpp
```

Used by [[components/parallel-task-queue|parallel-task-queue]] for the per-query MPMC task distribution queue. Capacity must be a power of 2; head/tail are atomic counters.

## `lockfree::bitmap`

```cpp
// CAS-based index allocation — used by tran::system to assign descriptor slots
lockfree_bitmap.hpp/cpp
```

Provides O(1) amortized slot allocation with a 64-bit CAS find-first-clear. Used internally by `lockfree::tran::system` to manage per-thread descriptor indices.

## Legacy API (`lock_free.c/h`)

```c
// Legacy — same concepts, C API
LF_HASH_TABLE    // hash table with chaining
LF_FREELIST      // node pool
LF_TRAN_SYSTEM   // global index manager
LF_TRAN_ENTRY    // per-thread descriptor

// Usage pattern:
LF_TRAN_ENTRY *tran_entry = lf_tran_start(&tran_system);
entry = lf_hash_find(&hash_table, tran_entry, &key);
lf_tran_end(&tran_system, tran_entry);
```

> [!warning] Do not mix generations
> Modern and legacy lock-free structures each have their own `tran::system` / `LF_TRAN_SYSTEM`. A transaction index from one system cannot be used with a structure from the other.

## Usage Rules

1. Always call `start_tran()` before accessing any structure; `end_tran()` after.
2. Do not hold a reference to a retrieved node across `end_tran()`.
3. `lockfree::hashmap::clear()` is **NOT lock-free** — it acquires an internal mutex. Only call during quiescent states.
4. Retired nodes are reclaimed lazily — memory may stay live briefly after `retire()`.

## Related

- [[components/base|base]] — parent hub
- [[components/parallel-task-queue|parallel-task-queue]] — uses `lockfree_circular_queue` for MPMC task dispatch
- [[components/parallel-query|parallel-query]] — uses atomic CAS via lock-free worker reservation
- [[components/memory-alloc|memory-alloc]] — node allocation for freelist blocks
- Source: [[sources/cubrid-src-base|cubrid-src-base]]
