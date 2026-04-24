---
type: component
parent_module: "[[modules/src|src]]"
path: "src/storage/page_buffer.c"
status: active
purpose: "In-memory page cache for all database volumes: LRU zone replacement, fix/unfix latch protocol, dirty tracking, WAL-safe flush, and double-write buffer integration"
key_files:
  - "page_buffer.c (~17K lines) — full implementation"
  - "page_buffer.h — public API, enums, PGBUF_WATCHER struct"
  - "double_write_buffer.hpp — DWB integration called at flush"
public_api:
  - "pgbuf_fix(thread_p, vpid, fetch_mode, requestmode, condition) -> PAGE_PTR"
  - "pgbuf_unfix(thread_p, pgptr)"
  - "pgbuf_set_dirty(thread_p, pgptr, free_page)"
  - "pgbuf_flush_with_wal(thread_p, pgptr)"
  - "pgbuf_flush_checkpoint(thread_p, flush_upto_lsa, prev_chkpt_redo_lsa, smallest_lsa, flushed_cnt)"
  - "pgbuf_flush_victim_candidates(thread_p, flush_ratio, time_tracker, stop)"
  - "pgbuf_ordered_fix / pgbuf_ordered_unfix (heap-ordered latch acquisition)"
  - "pgbuf_promote_read_latch(thread_p, pgptr_p, condition)"
  - "pgbuf_assign_private_lru / pgbuf_release_private_lru"
  - "pgbuf_peek_stats(...) — expose LRU zone counts"
  - "pgbuf_thread_variables_init(thread_p) — caches m_is_private_lru_enabled + m_holder_anchor on THREAD_ENTRY"
tags:
  - component
  - cubrid
  - storage
  - buffer-pool
related:
  - "[[components/storage|storage]]"
  - "[[components/double-write-buffer|double-write-buffer]]"
  - "[[components/transaction|transaction]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `page_buffer.c` — Buffer Pool

The page buffer is the single gate through which all page I/O passes. No component reads or writes a database volume directly — every access goes through `pgbuf_fix` / `pgbuf_unfix`. The implementation is ~17 K lines.

## Data Structures

### BCB (Buffer Control Block)

Each cached page has one `PGBUF_BCB` (internal, not in the public header). Key fields:

| Field | Type | Role |
|-------|------|------|
| `vpid` | `VPID` | Identity of the cached page |
| `zone_flags` | `atomic<int>` | Zone bits + BCB flags (dirty, flushing, victim, …) |
| `count_fix_and_avoid_dealloc` | `atomic<int>` | High 16 bits = fix count, low 16 bits = avoid-dealloc count |
| `lsa` | `LOG_LSA` | Page LSA at last dirty mark |
| `atomic_latch` | `std::atomic<uint64_t>` | Packed `{latch_mode: uint16_t, waiter_exists: uint16_t, fcnt: int32_t}` union — single-word CAS primitive; replaced the legacy `pthread_mutex_t` for all BCB state transitions |
| `latch_last_thread` | `THREAD_ENTRY *` (`SERVER_MODE`) | Last thread that acquired the latch — diagnostic aid for contention debugging |
| `lru_prev/next` | `PGBUF_BCB*` | Position within LRU list |
| `iopage_buffer` | `PGBUF_IOPAGE_BUFFER*` | Points to actual page data |

### PGBUF_WATCHER

Public struct used by callers that need to hold ordered latches on multiple pages (e.g. heap file traversal):

```c
struct pgbuf_watcher {
  PAGE_PTR pgptr;
  PGBUF_WATCHER *next, *prev;
  VPID group_id;          /* heap header VPID — defines latch ordering group */
  unsigned latch_mode:7;
  unsigned page_was_unfixed:1;
  unsigned initial_rank:4;
  unsigned curr_rank:4;
  /* debug fields in NDEBUG builds */
};
```

Ranks: `PGBUF_ORDERED_HEAP_HDR` < `PGBUF_ORDERED_HEAP_NORMAL` < `PGBUF_ORDERED_HEAP_OVERFLOW`.

## LRU Zone Architecture

> [!key-insight] Three-zone LRU — not simple LRU
> The buffer pool divides pages into three LRU zones. Victims are only selected from zone 3. This prevents a single large scan from evicting all hot OLTP pages.

```
Hot     Cold     Victim
Zone 1  Zone 2   Zone 3
  ─────────────────────►  (aging direction)
  │ no   │ no    │ yes  │  victimizable?
  │boost │ boost │ boost│  boosted on fix/unfix?
```

Zone membership per `PGBUF_ZONE` enum:

| Constant | Value | Role |
|----------|-------|------|
| `PGBUF_LRU_1_ZONE` | `1 << 16` | Hot zone — no victims, no boost needed |
| `PGBUF_LRU_2_ZONE` | `2 << 16` | Buffer zone — boosted back to hot if re-fixed |
| `PGBUF_LRU_3_ZONE` | `3 << 16` | Victim zone — dirty pages flushed then evicted |
| `PGBUF_VOID_ZONE` | `2 << 18` | Transient: page being read from disk or victimized |
| `PGBUF_INVALID_ZONE` | `1 << 18` | Not yet assigned to any list |

There can be up to 64 K separate LRU lists (`PGBUF_LRU_LIST_MAX_COUNT`). Private LRU lists are assigned per transaction (`pgbuf_assign_private_lru`) to improve locality. Activity thresholds determine when a private list is destroyed.

## BCB Flags

| Flag | Bit | Meaning |
|------|-----|---------|
| `PGBUF_BCB_DIRTY_FLAG` | `0x80000000` | Page has been modified |
| `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` | `0x40000000` | Being flushed — cannot victimize |
| `PGBUF_BCB_VICTIM_DIRECT_FLAG` | `0x20000000` | Assigned as victim to a sleeping thread |
| `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG` | `0x10000000` | Direct victim was re-fixed; re-assign |
| `PGBUF_BCB_MOVE_TO_LRU_BOTTOM_FLAG` | `0x08000000` | Move to zone 3 bottom on unfix (deallocated page) |
| `PGBUF_BCB_TO_VACUUM_FLAG` | `0x04000000` | Notify vacuum on next unfix |
| `PGBUF_BCB_ASYNC_FLUSH_REQ` | `0x02000000` | Async flush requested |

A BCB is an invalid victim candidate if any of `DIRTY | FLUSHING | VICTIM_DIRECT | INVALIDATE_DIRECT_VICTIM` is set.

## Fix / Unfix Protocol

```c
/* Standard pattern */
PAGE_PTR page = pgbuf_fix (thread_p, &vpid, OLD_PAGE,
                            PGBUF_LATCH_WRITE, PGBUF_UNCONDITIONAL_LATCH);
if (page == NULL) { /* error */ }
/* ... modify page ... */
pgbuf_set_dirty (thread_p, page, DONT_FREE);
pgbuf_unfix (thread_p, page);   /* or pgbuf_set_dirty_and_free */
```

`PAGE_FETCH_MODE` controls whether the fix expects an existing page or newly allocated one:

| Mode | Meaning |
|------|---------|
| `OLD_PAGE` | Existing page; validate and read from disk if not cached |
| `NEW_PAGE` | Newly allocated; skip disk read |
| `OLD_PAGE_IF_IN_BUFFER` | Only succeed if already in buffer |
| `OLD_PAGE_PREVENT_DEALLOC` | Pin in memory (increment avoid-dealloc counter) |
| `RECOVERY_PAGE` | Any state; used during redo/undo |

In debug builds `pgbuf_fix` maps to `pgbuf_fix_debug`, which records `caller_file`, `caller_line`, and `caller_func` for leak detection.

## Latch Promotion

A thread holding `PGBUF_LATCH_READ` can atomically upgrade to `PGBUF_LATCH_WRITE`:

```c
int rc = pgbuf_promote_read_latch (thread_p, &page, PGBUF_PROMOTE_ONLY_READER);
```

`PGBUF_PROMOTE_ONLY_READER` succeeds only if the caller is the sole reader. `PGBUF_PROMOTE_SHARED_READER` waits for all other readers to unfix first.

## Ordered Latch Protocol

For heap file traversal, multiple pages must be latched in a defined order to avoid deadlocks:

```c
PGBUF_WATCHER watcher;
PGBUF_INIT_WATCHER (&watcher, PGBUF_ORDERED_HEAP_NORMAL, &hfid);
pgbuf_ordered_fix (thread_p, &vpid, OLD_PAGE, PGBUF_LATCH_WRITE, &watcher);
/* ... */
pgbuf_ordered_unfix (thread_p, &watcher);
```

The system ensures pages in the same heap are always latched in `VPID` order within a rank group, automatically unfixing and re-fixing if ordering is violated.

## Flush Paths

| Function | Trigger | WAL-safe |
|----------|---------|----------|
| `pgbuf_flush_with_wal` | Manual flush of one page | Yes — flushes log first |
| `pgbuf_flush_victim_candidates` | Victim flush daemon | Yes |
| `pgbuf_flush_checkpoint` | Checkpoint thread | Yes — up to given LSN |
| `pgbuf_flush` | Direct forced flush | Caller responsible for WAL |
| `pgbuf_flush_all` | Full flush (volume close / backup) | No WAL guarantee |

The victim flush daemon runs in `SERVER_MODE` via `pgbuf_daemons_init()`. It calls `pgbuf_flush_victim_candidates` at a rate controlled by `pgbuf_flush_control_from_dirty_ratio`.

## Neighbor Flush Optimization

When flushing a victim, the buffer pool can flush up to `PRM_ID_PB_NEIGHBOR_FLUSH_PAGES` (default: configurable) adjacent pages in the same volume. Controlled by `PGBUF_NEIGHBOR_FLUSH_NONDIRTY`. This improves sequential write throughput.

## Double-Write Buffer Integration

Before any page reaches disk, it passes through the [[components/double-write-buffer|double-write buffer]]:
1. `pgbuf_flush_with_wal` calls `dwb_add_page(thread_p, io_page, &vpid, ...)`.
2. DWB accumulates pages into a block and syncs the DWB file first.
3. Only then does the page get written to its actual volume location.

This prevents torn writes across power failures.

## Statistics and Monitoring

`pgbuf_peek_stats` exposes counters for all three LRU zones, dirty page count, victim candidates, private LRU quota, and BCB waiter queues. Used by the `SHOW PAGE BUFFER STATUS` diagnostic command.

## Atomic latch model

Each BCB's mutable state (latch mode, waiter flag, fix count) lives in a single 64-bit `std::atomic<uint64_t>` called `atomic_latch`. A union overlay reinterprets the word as `{PGBUF_LATCH_MODE latch_mode (16 bits); uint16_t waiter_exists; int fcnt (32 bits)}`. Every state transition is a `compare_exchange_weak` retry loop — the mutex that used to guard this state is gone.

CAS primitive helpers (all file-local inlines in `page_buffer.c`):

| Primitive | Purpose |
|---|---|
| `set_latch(latch, mode)` | Rewrite `latch_mode`, preserve `fcnt` and `waiter_exists`. |
| `add_fcnt(latch, cnt)` | Atomically add to `fcnt`. |
| `set_latch_and_fcnt(latch, mode, cnt)` | Compound: rewrite mode AND set fcnt atomically. |
| `set_latch_and_add_fcnt(latch, mode, cnt)` | Compound: rewrite mode AND add to fcnt atomically. |
| `set_waiter_exists(latch, bool)` | Flip the waiter flag. |
| `get_fcnt / get_waiter_exists / get_latch / get_impl` | Acquire-ordered reads of individual fields or the whole union. |

Compound `_and_` setters update both fields in the same CAS — readers never observe a half-updated state.

### Lockfree RO fast path

For read-only fixes on pages already resident and not transitioning, a fast path entirely bypasses both the hash-anchor mutex and any BCB CAS contention:

- `pgbuf_lockfree_fix_ro` — called from `pgbuf_fix` when the caller wants a READ latch on `OLD_PAGE` / `OLD_PAGE_MAYBE_DEALLOCATED`. Walks the hash chain via `pgbuf_search_hash_chain_no_bcb_lock` (no anchor mutex), then a gate CAS verifies `{VPID matches, latch_mode ∈ {NO_LATCH, READ}, waiter_exists == false}` and atomically increments `fcnt`. On failure, falls back to the slow path.
- `pgbuf_lockfree_unfix_ro` — symmetric: CAS decrement `fcnt`. Falls back to slow path only if a waiter is present.

The VPID recheck inside the gate CAS defends against ABA — a BCB being recycled to a different VPID between the lookup and the CAS.

### `PGBUF_THREAD_HAS_PRIVATE_LRU` is on the hot path

The predicate is consulted on every page fix. To minimize per-fix load overhead it reads a single cached bool `thread_p->m_is_private_lru_enabled` (on `cubthread::entry`) rather than recomputing `quota_enabled && private_lru_index != -1`.

The cached bool is populated by `pgbuf_thread_variables_init(thread_p)`, called from three thread-reassignment sites:

- `css_server_task::execute` — after `private_lru_index = session_get_private_lru_idx(session_p)`.
- `vacuum_master_task::execute` — after boot guard, on master wake-up.
- `session_set_conn_entry_data` — session attach / reconnect mid-task.

**Contract**: any code that writes `thread_p->private_lru_index` **must** call `pgbuf_thread_variables_init` to refresh the cache. Missing a call leaves the bool stale and routes subsequent page fixes to the wrong LRU.

The same init function also caches `thread_p->m_holder_anchor = &pgbuf_Pool.thrd_holder_info[thread_p->index]` to avoid per-fix array indexing. For daemons that never transit the three init sites (vacuum workers, log flush, checkpoint, DWB, CDC), four holder-accessor sites in `page_buffer.c` lazily bind the pointer on first use.

## Gotchas

> [!warning] Max simultaneously fixed pages per thread
> A single thread may hold at most `PGBUF_MAX_PAGE_FIXED_BY_TRAN` (64) pages fixed simultaneously. Exceeding this causes an assertion failure in debug builds.

> [!warning] Vacuum workers ignored for zone boosting
> `PGBUF_VACUUM_SHOULD_IGNORE_UNFIX` returns `true` for vacuum threads. Vacuum page access deliberately does not boost pages to hot zones — this prevents vacuum from polluting the buffer pool with pages that are not normally hot.

## Related

- Parent: [[components/storage|storage]]
- [[components/double-write-buffer]] — flush destination for dirty pages
- [[components/transaction]] — WAL LSN management, checkpoint coordination
- [[components/file-manager]] — `file_io.c` performs actual pread/pwrite
