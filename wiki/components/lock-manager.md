---
type: component
parent_module: "[[components/transaction|transaction]]"
path: "src/transaction/lock_manager.c, lock_manager.h"
status: active
purpose: "Hierarchical object lock manager: acquire, release, escalation, instant locks"
key_files:
  - "lock_manager.h (LK_RES, LK_ENTRY, LK_COMPOSITE_LOCK, lock grant/wait result codes)"
  - "lock_manager.c (~15K lines: acquire path, escalation, deadlock daemon)"
public_api:
  - "lock_object(thread_p, oid, class_oid, lock, cond_flag) → int"
  - "lock_object_wait_msecs(thread_p, oid, class_oid, lock, cond_flag, wait_msecs)"
  - "lock_scan(thread_p, class_oid, cond_flag, class_lock)"
  - "lock_unlock_object(thread_p, oid, class_oid, lock, force)"
  - "lock_unlock_all(thread_p)"
  - "lock_demote_class_lock(thread_p, oid, lock, ex_lock)"
  - "lock_has_xlock(thread_p) → bool"
  - "lock_initialize() / lock_finalize()"
tags:
  - component
  - cubrid
  - locking
  - transaction
  - concurrency
related:
  - "[[components/transaction|transaction]]"
  - "[[components/deadlock-detection|deadlock-detection]]"
  - "[[components/mvcc|mvcc]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/btree|btree]]"
  - "[[components/heap-file|heap-file]]"
created: 2026-04-23
updated: 2026-04-23
---

# Lock Manager

`src/transaction/lock_manager.c` (~15K lines) manages all logical transaction locks in CUBRID. It is distinct from page **latches** (short-term, physical) held by [[components/page-buffer|page-buffer]].

> [!warning] Latch vs lock distinction
> Page latches (`pgbuf_fix` latch modes) protect in-memory buffer consistency for the duration of an operation. Transaction locks (`lock_object`) protect logical objects across transaction boundaries. B-tree and heap operations hold both simultaneously — acquiring the wrong type causes subtle deadlocks.

## Key Structures

### `LK_RES` — Lock resource (one per locked object)

```c
struct lk_res {
  LK_RES_KEY         key;                 /* {type, oid, class_oid} */
  LOCK               total_holders_mode;  /* supremum of granted locks */
  LOCK               total_waiters_mode;  /* supremum of waiting locks */
  LK_ENTRY          *holder;             /* granted-lock list */
  LK_ENTRY          *waiter;            /* waiting-lock list */
  LK_ENTRY          *non2pl;            /* non-2PL (instant lock) list */
  pthread_mutex_t    res_mutex;
  LK_RES            *hash_next;         /* lock-free hash chain */
};
```

Resources are hashed by OID using a lock-free hash map (`thread_lockfree_hash_map`). The hash function for temp OIDs uses `-(oid->pageid)` to avoid collision with permanent OIDs.

### `LK_ENTRY` — Per-transaction lock entry

```c
struct lk_entry {
  struct lk_res *res_head;      /* back-pointer to resource */
  THREAD_ENTRY  *thrd_entry;
  int            tran_index;
  LOCK           granted_mode;  /* currently granted lock strength */
  LOCK           blocked_mode;  /* lock mode being waited for */
  int            count;         /* number of (re)lock requests */
  LK_ENTRY      *next;          /* next in resource holder/waiter list */
  LK_ENTRY      *tran_next;     /* next in transaction's held-lock list */
  LK_ENTRY      *tran_prev;
  LK_ENTRY      *class_entry;  /* pointer to owning class lock entry */
  int            ngranules;    /* number of finer-grained locks under this */
  int            instant_lock_count;
};
```

### `LOCK_RESOURCE_TYPE` — Resource granularity

| Type | Description |
|------|-------------|
| `LOCK_RESOURCE_ROOT_CLASS` | The catalog root (`OID_ROOT_CLASS_OID`) |
| `LOCK_RESOURCE_CLASS` | A table/class |
| `LOCK_RESOURCE_INSTANCE` | A row / instance |

## Lock Modes

CUBRID uses 8 lock modes (compatible matrix enforced at acquire time):

| Mode | Abbrev | Usage |
|------|--------|-------|
| `NULL_LOCK` | NL | No lock |
| `IS_LOCK` | IS | Intent shared — plan to read rows |
| `IX_LOCK` | IX | Intent exclusive — plan to write rows |
| `S_LOCK` | S | Shared — read |
| `SIX_LOCK` | SIX | Shared + intent exclusive |
| `X_LOCK` | X | Exclusive — write |
| `SCH_S_LOCK` | SCH-S | Schema shared — protect schema during read |
| `SCH_M_LOCK` | SCH-M | Schema exclusive — DDL |

Lock upgrade path: `IS → IX → SIX → X` and `IS → S → SIX → X`.

## Grant / Wait Result Codes

```c
LK_GRANTED                     // lock acquired immediately
LK_NOTGRANTED                  // generic failure
LK_NOTGRANTED_DUE_ABORTED      // transaction aborted (deadlock victim)
LK_NOTGRANTED_DUE_TIMEOUT      // wait_msecs expired
LK_NOTGRANTED_DUE_ERROR        // internal error
LK_GRANTED_PUSHINSET_LOCKONE   // composite lock: pushed to set, locked one
LK_GRANTED_PUSHINSET_RELOCKALL // composite lock: pushed to set, relock all
```

Wait constants: `LK_ZERO_WAIT = 0`, `LK_INFINITE_WAIT = -1`, `LK_FORCE_ZERO_WAIT = -2` (timeout without error).

## Lock Acquire Path

```
lock_object(thread_p, oid, class_oid, lock, cond_flag)
  │
  ├── Find or create LK_RES in hash map
  │     (lock_free hash: OID → LK_RES*)
  │
  ├── Check compatibility with total_holders_mode
  │     ┌── Compatible → insert LK_ENTRY into holder list; return LK_GRANTED
  │     └── Not compatible:
  │           ├── cond_flag == LK_COND_LOCK → return LK_NOTGRANTED immediately
  │           └── cond_flag == LK_UNCOND_LOCK:
  │                 ├── Insert LK_ENTRY into waiter list
  │                 ├── Thread suspends (pthread_cond_wait)
  │                 ├── Deadlock daemon wakes periodically; may abort this tran
  │                 └── On wake: retry compatibility check
  │
  └── Also acquire class (intent) lock if instance lock requested
```

> [!key-insight] Younger-transaction priority
> When a deadlock is detected, the deadlock daemon selects the **youngest** transaction (highest `tranid`) as the victim, using `LK_ISYOUNGER(young_tranid, old_tranid)` macro. This biases toward aborting more recently started work, preferring to keep long-running transactions alive.

## Lock Escalation

When a transaction holds more row locks on a class than the system threshold (`LK_COMPOSITE_LOCK` / `PRM_ID_LK_ESCALATION_AT`), `lock_manager.c` escalates to a class-level lock:
1. Acquire class `X_LOCK` (or appropriate mode).
2. Release all per-instance locks for that class.
3. Set `ngranules` counter on the class `LK_ENTRY`.

`LK_LOCKCOMP` / `LK_LOCKCOMP_CLASS` track the in-progress escalation state per transaction.

## Instant Locks

`lock_start_instant_lock_mode()` enables a mode where locks are acquired but immediately released after use (not held until commit). Used by certain catalog operations. `lock_stop_instant_lock_mode()` disables it and optionally releases all instant locks.

## Composite Locks (`LK_COMPOSITE_LOCK`)

Batch-lock API for bulk DML:
1. `lock_initialize_composite_lock()` — allocate composite lock.
2. `lock_add_composite_lock(comp_lock, oid, class_oid)` — accumulate OIDs.
3. `lock_finalize_composite_lock()` — acquire all locks at once (with escalation check).
4. `lock_abort_composite_lock()` — release on error.

## Deadlock Detection

The deadlock detection daemon runs inside `lock_manager.c` (not the legacy `wait_for_graph.c`, which is `#ifdef ENABLE_UNUSED_FUNCTION`). See [[components/deadlock-detection|deadlock-detection]] for details.

## Replication / HA Lock Support

`lock_reacquire_crash_locks()` re-grants locks held at the time of a crash during HA failover. `lock_unlock_all_shared_get_all_exclusive()` atomically upgrades all shared locks to exclusive during promotion.

## Related

- Parent: [[components/transaction|transaction]]
- [[components/deadlock-detection|deadlock-detection]] — deadlock detection algorithm
- [[components/mvcc|mvcc]] — MVCC supplements (but does not replace) locking for DML
- [[components/page-buffer|page-buffer]] — page latches (short-term, distinct from transaction locks)
- [[components/btree|btree]] — acquires row locks during index scan; uses `lock_object` with `S_LOCK` / `X_LOCK`
- [[components/heap-file|heap-file]] — calls `lock_object` for instance locks before heap operations
- Source: [[sources/cubrid-src-transaction]]
