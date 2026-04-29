---
address: c-000006
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/parallel/px_scan/px_scan_input_handler_index.{cpp,hpp}, px_scan_slot_iterator_index.{cpp,hpp}, px_scan.cpp (manager) — branch parallel_scan_all"
status: branch-wip
purpose: "Parallel scan input handler + slot iterator for SCAN_TYPE::INDEX: latch-coupled deferred root descent, mutex-guarded leaf chain advance, and the four interlocking subsystem invariants required to ship parallel index scan on partitioned tables (CBRD-26722)"
key_files:
  - "src/query/parallel/px_scan/px_scan_input_handler_index.hpp (m_btid / m_btid_int / m_indx_info, public init/initialize/get_indx_info, m_leaf_mutex)"
  - "src/query/parallel/px_scan/px_scan_input_handler_index.cpp (init_on_main minimal, lazy descend_to_first_leaf, leaf-chain advance under mutex)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator_index.hpp (per-worker key cursor, range-cursor state)"
  - "src/query/parallel/px_scan/px_scan_slot_iterator_index.cpp (1026 LOC: collect_oid_callback, fence-key skip, covering vs heap-fetch, midxkey reject)"
  - "src/query/parallel/px_scan/px_scan_task.cpp (worker initialize: BTID restore for partitioned-table case)"
  - "src/query/parallel/px_scan/px_scan.cpp (manager<RT, INDEX>::open/close, scan_try_promote_parallel_index_scan, parallel_index_scan_pending struct)"
  - "src/query/scan_manager.h (PARALLEL_INDEX_SCAN_ID — superset of INDX_SCAN_ID at identical offsets, paired offsetof asserts)"
  - "src/query/scan_manager.c (scan_open_parallel_index_scan deferred promotion; scan_close_parallel_index_scan trace_storage hand-off)"
  - "src/query/query_executor.c (qexec_init_next_partition: per-partition live indexptr update + final-iteration parent-class re-open)"
  - "src/query/query_dump.c (dump branches now accept S_PARALLEL_INDEX_SCAN || S_INDX_SCAN)"
public_api:
  - "input_handler_index::init_on_main(thread_p, indx_info) — minimal: BTID_COPY + m_indx_info store"
  - "input_handler_index::initialize(thread_p, ...) — per-worker entry"
  - "input_handler_index::descend_to_first_leaf(thread_p) — lazy first-worker root→leaf descent, populates m_btid_int via btree_glean_root_header_info"
  - "input_handler_index::get_next_leaf_with_fix(thread_p, out_page) — m_leaf_mutex-guarded advance"
  - "input_handler_index::get_indx_info() → INDX_INFO * — safe post-init_on_main"
  - "input_handler_index::get_btid_int() → BTID_INT * — UNSAFE before first descend_to_first_leaf"
  - "slot_iterator_index::set_page / next_qualified_slot / finalize"
  - "scan_open_parallel_index_scan(...) — deferred promotion, stashes parallel_index_scan_pending in isid"
  - "scan_try_promote_parallel_index_scan(...) — main-thread promote attempt, falls back to S_INDX_SCAN cleanly"
  - "scan_close_parallel_index_scan(...) — manager close + trace_storage hand-off"
tags:
  - component
  - cubrid
  - parallel
  - parallel-scan
  - parallel-index-scan
  - btree
  - partition
  - branch-wip
  - CBRD-26722
related:
  - "[[components/parallel-heap-scan|parallel-heap-scan]]"
  - "[[components/parallel-list-scan|parallel-list-scan]]"
  - "[[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/btree|btree]]"
  - "[[components/partition-pruning|partition-pruning]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/xasl|xasl]]"
  - "[[prs/PR-7062-parallel-scan-all-types|PR #7062]]"
  - "[[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables|CBRD-26722 knowledge dump]]"
created: 2026-04-29
updated: 2026-04-29
---

# `parallel_scan::input_handler_index` / `slot_iterator_index` — Parallel Index Scan

> [!warning] Branch-WIP, not in baseline
> All file paths and line numbers on this page reflect branch `parallel_scan_all` (head `7fdb82099`, [CBRD-26722], [[prs/PR-7062-parallel-scan-all-types|PR #7062]] OPEN). The `src/query/parallel/px_scan/` directory does **not** exist in baseline `0be6cdf6` — the heap-only predecessor `src/query/parallel/px_heap_scan/` (see [[components/parallel-heap-scan-input-handler]]) is what's on disk at baseline. This page captures forward-looking design and the four-invariant fix for parallel index scan on partitioned tables. It will be reconciled into a canonical `parallel-scan-input-handler-index` page when PR #7062 merges.

The index-scan variant of the generalised `parallel_scan::manager<RESULT_TYPE, SCAN_TYPE>` template (`SCAN_TYPE::INDEX`). Its job: turn one B+tree range scan into N parallel readers that share a leaf cursor under a mutex, descend the root once (deferred), and propagate per-partition BTIDs through the XASL clone boundary.

Sibling variants in the same template family: `input_handler_heap` (renamed from `input_handler_ftabs`, see [[components/parallel-heap-scan-input-handler]]), `input_handler_list` (see [[components/parallel-list-scan]]).

## CBRD-26722 — parallel index scan on partitioned tables, in four commits

Parallel index scan was unconditionally disabled on partitioned tables. Naive guard relaxation crashed with `ER_PT_EXECUTE(-495)`. The shipped fix is **four commits** because the feature crossed four different invariants — struct layout, type-machine state, per-partition BTID propagation through the XASL stream, and dump-path type discrimination.

```
67e0eb852  C1  PARALLEL_INDEX_SCAN_ID superset of INDX_SCAN_ID
1867903c0  C2  Relax guard at px_scan.cpp:1319 (parent-class only)
9185c1aae  C3  Restore partition BTID in worker initialize
7fdb82099  C4  Dump parallel-index trace_storage on S_INDX_SCAN type
```

Source: [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables|CBRD-26722 knowledge dump]] captures the analysis-loop history; this page is the structural distillation.

---

## Invariant 1 — `PARALLEL_INDEX_SCAN_ID` must be a layout superset of `INDX_SCAN_ID`

`SCAN_ID::s` is a polymorphic union over 15 scan types (see [[components/scan-manager]] § "SCAN_ID: The Polymorphic Scan Handle"). For a single scan that promotes from serial → parallel and reopens across partitions, `scan_id->type` flips between `S_INDX_SCAN` and `S_PARALLEL_INDEX_SCAN` repeatedly. Each flip changes which union member is "live."

Pre-fix layout (broken):
```
struct indx_scan_id            { ~50 fields; ~hundreds of bytes }
struct parallel_index_scan_id  { 3 fields:  result_type / manager / trace_storage; 24 bytes }
```

The two structs **share storage** by union design. Writing 3 pisid fields at offsets 0..23 corrupts the first 24 bytes of isid. After one promote, the first 24 bytes of isid (`indx_info`, `bt_type`, `bt_num_attrs`...) are pisid garbage; the rest is stale isid. Any subsequent code that reads through the isid view dereferences invalid pointers.

> [!key-insight] PARALLEL_HEAP_SCAN_ID does NOT have this hazard
> `PARALLEL_HEAP_SCAN_ID` is defined as a **superset** of `HEAP_SCAN_ID` — every hsid field mirrored at identical offsets, parallel-only fields appended. Union flip preserves layout. The original `PARALLEL_INDEX_SCAN_ID` deviated from this pattern (3 fields at offsets 0..23) and corrupted the isid view on every flip. Fix: reshape pisid to mirror all 50+ INDX_SCAN_ID fields at identical offsets, then append `result_type / manager / trace_storage`.

**C1 (`67e0eb852`)** pins the invariant with paired `offsetof` asserts so future drift fails at compile time:

```cpp
// src/query/scan_manager.h:170-313
static_assert (offsetof(INDX_SCAN_ID, indx_info)        == offsetof(PARALLEL_INDEX_SCAN_ID, indx_info));
static_assert (offsetof(INDX_SCAN_ID, bt_scan)          == offsetof(PARALLEL_INDEX_SCAN_ID, bt_scan));
static_assert (offsetof(INDX_SCAN_ID, scan_cache)       == offsetof(PARALLEL_INDEX_SCAN_ID, scan_cache));
static_assert (offsetof(INDX_SCAN_ID, caches_inited)    == offsetof(PARALLEL_INDEX_SCAN_ID, caches_inited));
static_assert (offsetof(INDX_SCAN_ID, scancache_inited) == offsetof(PARALLEL_INDEX_SCAN_ID, scancache_inited));
static_assert (offsetof(INDX_SCAN_ID, indx_cov)         == offsetof(PARALLEL_INDEX_SCAN_ID, indx_cov));
static_assert (offsetof(INDX_SCAN_ID, multi_range_opt)  == offsetof(PARALLEL_INDEX_SCAN_ID, multi_range_opt));
static_assert (offsetof(INDX_SCAN_ID, iss)              == offsetof(PARALLEL_INDEX_SCAN_ID, iss));
static_assert (offsetof(INDX_SCAN_ID, iscan_oid_order)  == offsetof(PARALLEL_INDEX_SCAN_ID, iscan_oid_order));
static_assert (offsetof(INDX_SCAN_ID, parallel_pending) == offsetof(PARALLEL_INDEX_SCAN_ID, parallel_pending));
static_assert (offsetof(PARALLEL_INDEX_SCAN_ID, result_type) >= sizeof(INDX_SCAN_ID),
               "pisid parallel-only fields must follow all isid fields");
```

> [!gap] GCC `offsetof` warning workaround
> GCC complains `offsetof on non-standard-layout type` for `SCAN_PRED`-containing structs — this is conditionally-supported but well-defined. Suppress with `pragma push/pop`. If you remove or relocate the asserts, the compiler will silently accept future layout drift and the corruption returns.

---

## Invariant 2 — `manager::close()` is the canonical destruction site, NOT explicit destroy + free

```cpp
// px_scan.cpp:2169-2187
template <RESULT_TYPE result_type, SCAN_TYPE ST>
int manager<result_type, ST>::close()
{
  THREAD_ENTRY *thread_p = m_thread_p;
  if constexpr (ST == SCAN_TYPE::INDEX) {
    m_scan_id->s.pisid.manager = nullptr;          // null the pointer FIRST
  } else if constexpr (ST == SCAN_TYPE::LIST) {
    m_scan_id->s.pllsid_parallel.manager = nullptr;
  } else {
    m_scan_id->s.phsid.manager = nullptr;
  }
  this->~manager();                                // destroy
  db_private_free (thread_p, this);                // free
  return NO_ERROR;
}
```

> [!key-insight] close() does destructor + free; not "release resources but keep object alive"
> If you call `close()` twice → use-after-free. If you call `~manager()` after `close()` → double-free. The dispatch chain `scan_close_scan → scan_close_parallel_index_scan → manager_p->close()` already handles destruction at partition transition (`qexec_init_next_partition:8795`). Adding an explicit `~manager() + db_private_free` after that is a double-free bug.
>
> This is a common review trap: the close/destroy split looks asymmetric vs. the open()/~manager() pairing one would expect from RAII C++. Don't add a "missing destroy hook" — verify close() first.

---

## Invariant 3 — XASL stream carries compile-time BTID; per-partition runtime updates don't propagate to clones

[[components/xasl]] streams are pointer-as-byte-offset with a 256-bucket visited-ptr hashtable, generated client-side and transmitted to server. The serialized form is **frozen at compile time**.

`qexec_init_next_partition` (`query_executor.c:9073`) updates the **live in-memory** `spec->indexptr->btid` per partition before opening the scan:
```c
btid = spec->curent->btid;        // current partition's BTID
...
spec->indexptr->btid = btid;      // live update
```

Main-thread `scan_open_index_scan(thread_p, ..., spec->indexptr, ...)` reads the live indexptr → uses partition's BTID. **Works for the main thread.**

But parallel workers do `clone_xasl(thread_ref)` which **re-deserializes from the XASL stream** to get a fresh per-worker copy. The stream still carries the original parent-class BTID. Workers' cloned `spec->indexptr->btid` therefore points to the parent partitioned class's B-tree root, not the active partition's.

**Symptom:** each worker calls `scan_open_index_scan` → `pgbuf_fix(Root_vpid)` on the parent class root page. **CUBRID partitioned parent classes have no usable B-tree** — `pgbuf_fix` returns NULL. The function at `scan_manager.c:3107-3111` returns `ER_FAILED` **without setting any error code** (no `er_set` call before the return):

```c
Root = pgbuf_fix (thread_p, &Root_vpid, OLD_PAGE, PGBUF_LATCH_READ, PGBUF_UNCONDITIONAL_LATCH);
if (Root == NULL)
  {
    return ER_FAILED;     // silent — no er_set
  }
```

Worker error path: `move_top_error_message_to_this()` finds an empty thread-local error context, `m_err_messages.m_error_messages[0]` is empty, `manager.next()` returns `S_ERROR` with `errid=0`. Caller `er_clear()`s upstream → `qexec_execute_mainblock` wrapper at `query_executor.c:16581` sees `stat != NO_ERROR && er_errid() == NO_ERROR` → emits generic `ER_PT_EXECUTE(-495)`.

> [!key-insight] Silent NULL → wrapped generic error pattern
> This silent-failure → wrapped-generic-error pattern was the entire reason the analysis loop took multiple rounds before C3 landed. When `pgbuf_fix` returns NULL the function bails without `er_set`; downstream the only signal that survives is the wrapper at `qexec_execute_mainblock:16581`. Diagnostic technique: file-based `fprintf` logging at suspected sites — server stderr is captured but not always visible; `er_set` doesn't fire so server log is empty.

**C3 (`9185c1aae`)** — after `clone_xasl`, override the worker's cloned `spec->indexptr->btid` with the manager's `input_handler` `indx_info` (captured at promote time):

```cpp
// px_scan_task.cpp:130-145
INDX_INFO *part_indx_info = m_input_handler->get_indx_info();
if (part_indx_info != nullptr)
  BTID_COPY (&spec->indexptr->btid, &part_indx_info->btid);
```

> [!warning] Stream is compile-time frozen; runtime in-memory edits do NOT survive clone
> Any path that mutates `spec->indexptr` (or any other XASL field) on the main thread to "communicate" runtime state to workers is broken under the parallel framework. Workers see the stream-derived value, not the live mutation. Communicate runtime state through the manager / input_handler instead — those are constructed on the main thread before workers spawn, and worker access is via `m_input_handler->get_*()` getters.

---

## Invariant 4 — `m_btid_int.sys_btid` is NULL until first worker descends

[[prs/PR-7062-parallel-scan-all-types|PR #7062]] commit `0f8a107bb` ("Latch-couple px_scan index descent and harden parallel paths") changed the root descent strategy. Previously each worker independently descended root → leaf. After the commit, only the FIRST worker descends; later workers re-fix the leaf VPID directly. This blocks concurrent splits between VPID resolve and re-fix.

Implementation: `input_handler_index::init_on_main` (`px_scan_input_handler_index.cpp:42`) is now minimal — just `BTID_COPY(&m_btid, &indx_info->btid)` plus `m_indx_info = indx_info`. **It does NOT pgbuf_fix the root.** The actual descent moved to `input_handler_index::descend_to_first_leaf` (`px_scan_input_handler_index.cpp:56-90`) which is called lazily by the first worker.

| Field | When populated | Safe to read post-promote? |
|---|---|---|
| `m_btid` (raw `BTID` value) | `init_on_main` (main thread) | yes |
| `m_indx_info` (pointer to live INDX_INFO) | `init_on_main` (main thread) | yes |
| `m_btid_int.sys_btid` | first worker's `descend_to_first_leaf` | **no** — NULL until then |
| `m_btid_int.{key_type, ...}` | populated by `btree_glean_root_header_info` inside `descend_to_first_leaf` | **no** — NULL until then |

> [!key-insight] BTID accessor lifecycle on parallel index scan
> When adding a new "BTID accessor" for parallel index scan, prefer one of:
>
> 1. `m_input_handler->get_indx_info()->btid` — safe post-`init_on_main`
> 2. A new getter `get_btid()` returning `&m_btid` — equivalent
>
> **Don't** dereference `m_btid_int.sys_btid` outside the descend-then-read code path. The C3 fix uses `get_indx_info()` (existing getter for `m_indx_info`) precisely because `get_btid_int()->sys_btid` is NULL at the worker `initialize` moment.

---

## Invariant 5 — Parent-class re-open rolls type back to `S_INDX_SCAN`; dump path must accept both types

`qexec_init_next_partition` final-iteration semantics:
- After last partition, `spec->curent` transitions to NULL.
- Re-enter the switch with `spec->curent == NULL` → set `class_oid` / `hfid` / `btid` back to parent class.
- `ACCESS_METHOD_INDEX` branch calls `scan_open_index_scan` (line 9075) with parent BTID.
- `scan_open_index_scan:3095` resets `scan_id->type = S_INDX_SCAN`.
- Then calls `scan_open_parallel_index_scan` (line 9098) — guard short-circuits at parent-class first call (`curent==NULL` branch from C2 fix).
- Returns `S_END` at line 9166 (`curent == NULL`).

By the time `query_dump` runs (after all scans done), `scan_id->type` is `S_INDX_SCAN`, NOT `S_PARALLEL_INDEX_SCAN`. The pisid superset (Invariant 1) keeps `pisid.trace_storage` valid through the type flip — but the dump branches were checking type only:

```c
// query_dump.c — pre-fix
else if (spec->s_id.type == S_PARALLEL_INDEX_SCAN)
{
  if (spec->s_id.s.pisid.trace_storage) { dump_stats_text(...); ... }
}
```

→ `trace_storage` holds populated stats but check fails → `(parallel workers: N, ...)` line missing from output.

> [!key-insight] HEAP path does not have this hazard
> The HEAP path is asymmetric:
>
> 1. HEAP partitioned doesn't call a separate `scan_open_heap_scan` first. Only `scan_open_parallel_heap_scan` is called, which contains type management internally.
> 2. The HEAP dump check at `query_dump.c:3540` already has the OR pattern: `S_PARALLEL_HEAP_SCAN || S_HEAP_SCAN`.
>
> INDEX path needed the same OR pattern. **C4 (`7fdb82099`)** changes `type == S_PARALLEL_INDEX_SCAN` to `type == S_PARALLEL_INDEX_SCAN || type == S_INDX_SCAN` at both `query_dump.c:3093` (json) and `:3553` (text). The inner `if (pisid.trace_storage != NULL)` check still gates true-serial scans out.

---

## Cross-cutting: `trace_storage` life-cycle for parallel-index × partition

For HEAP and LIST, `trace_storage` is straightforward. For parallel-index × partition, the storage actually leaks per partition (small):

```
partition N scan:
  1. workers compute partial aggregate
  2. main reads result; scan_next returns S_END
  3. scan_close_parallel_index_scan: lazy-alloc trace_storage if NULL → add_stats(manager.trace_handler)
  4. manager.close(): destroys manager, pisid.manager = nullptr
  5. query_executor.c:8843 set_last_partition_stats(&spec->parts[N]->scan_stats)
     (COPIES m_stats_last into per-partition slot; accumulator state preserved)

partition N+1 entry (qexec_init_next_partition):
  6. ACCESS_METHOD_INDEX branch
  7. isid cleanup, scan_open_index_scan (sets type = S_INDX_SCAN, partition N+1 BTID)
  8. scan_open_parallel_index_scan stashes new pending
  9. scan_start_scan → scan_try_promote
 10. Line 1560: scan_id->s.pisid.trace_storage = nullptr;   ← orphans partition N's accumulator
 11. New manager open → workers later → repeat from step 1
```

Each partition's accumulator is orphaned at step 10 (small leak — `sizeof(accumulative_trace_storage)`). The aggregate `parallel workers: N` line shows MIN..MAX across the LAST partition's workers only (e.g., `index time: 0..0` if last partition is empty). This is a known cosmetic limitation — aggregate SCAN-level counts and PARTITION lines are correct.

> [!gap] Cross-partition trace aggregation
> If aggregating across partitions is required, the fix is structural: hoist `trace_storage` out of pisid into `SCAN_ID` itself, OR avoid nulling it at `scan_try_promote` line 1560 and let the canonical destruction (query_dump.c) handle freeing once at end of query.

---

## Diagnosis playbook (for similar future bugs)

When you see `ER_PT_EXECUTE(-495)` with empty underlying error:

1. **Find the qp_xasl_line** in the err message (`Query execution failure #NNNNN`).
2. Trace `ER_PT_EXECUTE` wrapper at `qexec_execute_mainblock` ~line 16581 — it fires when `stat != NO_ERROR && er_errid() == NO_ERROR`.
3. The actual error was `er_clear()`'d upstream. Common `er_clear` sites:
   - `scan_try_promote_parallel_index_scan` promote-fail (`px_scan.cpp:1547`) → fall back to `S_INDX_SCAN`
   - many in `query_executor.c`
4. Use file-based `fprintf` logging (NOT `er_set` / NOT stderr) to log exact `err_code` from suspect sites. Server stderr fd is owned by parent process and may be invisible.
5. For silent NULL returns from `pgbuf_fix`, check if main thread is still holding latch (page-buffer mode mismatch), or if the page/file VPID is invalid.
6. `er_print_callstack` can dump callstack to err log — useful for tracing destruction order.

For dual-mode (parallel/serial) functions, check the sequence:
- `manager::close()` already destroys; never call `~manager()` after it.
- `scan_id->type` is a state-machine variable; flips multiple times per query for parallel-on-partition.
- Worker's view of `spec->indexptr` is a **clone**, not the live in-memory object; runtime in-memory edits to the indexptr don't propagate to workers.

---

## File map (verified at branch HEAD `7fdb82099`)

| Path | Role |
|---|---|
| `src/query/scan_manager.h:170-313` | `parallel_index_scan_id` superset definition + paired offsetof asserts (C1) |
| `src/query/parallel/px_scan/px_scan.cpp:1319-1322` | Guard relax — parent-class only (C2) |
| `src/query/parallel/px_scan/px_scan.cpp:1547` | `er_clear` site at promote-fail fallback |
| `src/query/parallel/px_scan/px_scan.cpp:1560` | `scan_id->s.pisid.trace_storage = nullptr` — per-partition orphan point |
| `src/query/parallel/px_scan/px_scan.cpp:2169-2187` | `manager::close()` destroy + free |
| `src/query/parallel/px_scan/px_scan_input_handler_index.cpp:42-55` | Minimal `init_on_main` (post-latch-couple) |
| `src/query/parallel/px_scan/px_scan_input_handler_index.cpp:56-90` | Lazy `descend_to_first_leaf` (sets sys_btid via `btree_glean_root_header_info`) |
| `src/query/parallel/px_scan/px_scan_task.cpp:130-145` | Worker BTID restore in `task::initialize` (C3) |
| `src/query/query_executor.c:9073` | Per-partition BTID write to live indexptr |
| `src/query/query_executor.c:8794-8847` | Partition transition: `scan_close`, `set_last_partition_stats` |
| `src/query/query_executor.c:16581` | `ER_PT_EXECUTE(-495)` wrapper at qexec_execute_mainblock |
| `src/query/query_dump.c:3093, 3553` | Dump branches now accept `S_PARALLEL_INDEX_SCAN \|\| S_INDX_SCAN` (C4) |
| `src/query/scan_manager.c:3107-3111` | Silent NULL return from `pgbuf_fix` (no `er_set`) |

---

## Acceptance criteria evidence (run on `7fdb82099`)

```
P region parallel workers: 5    (G1: ≥5)
P region PARTITION lines:  30   (G2: per-surviving-partition)
P/P-cmp result match:      P1✓ P2✓ P3✓ P4✓ P5✓
H region parallel workers: 2    (regression: HEAP partitioned still works)
L region parallel workers: 4    (regression: TEMP)
I region parallel workers: 8    (regression: INDEX non-partition)
F region parallel workers: 0    (correctly serial fallback)
ER_PT_EXECUTE(-495):       0
```

---

## Differences from `parallel-list-scan` (sibling branch-WIP page)

| Aspect | `input_handler_list` | `input_handler_index` |
|---|---|---|
| Distribution unit | sectors of `QFILE_LIST_ID` | leaves of B+tree |
| Partition strategy | static slice `[start, end)` per worker | shared cursor under `m_leaf_mutex`; one leaf per acquire |
| Root descent | n/a (no tree) | deferred — first live worker via `descend_to_first_leaf` |
| Membuf fast path | lazy CAS-claim (first live worker) | n/a |
| Partitioned-table propagation | n/a (lists don't partition the same way) | `clone_xasl` boundary fix via `m_input_handler->get_indx_info()` |
| Type-flip hazard | none reported | the four invariants above (C1–C4) |

---

## Constraints

- **Threading**: `init_on_main` runs single-threaded on main; `initialize` / `descend_to_first_leaf` / leaf advance run per-worker.
- **Build-mode**: SERVER_MODE + SA_MODE only.
- **Lifetime**: handler is `db_private_alloc`'d + placement-new constructed by `manager::open`; explicit destructor + `db_private_free` on teardown via `manager::close()` (see Invariant 2).
- **Snapshot mode**: `XASL_SNAPSHOT × INDEX` is **blocked by the checker** (`px_scan_checker.cpp`); only `MERGEABLE_LIST` and `BUILDVALUE_OPT` modes use this handler.
- **Promote contract**: `scan_open_parallel_index_scan` does NOT eagerly open the manager. It stashes a `parallel_index_scan_pending` struct in `scan_id->s.isid.parallel_pending`; `scan_start_scan` later calls `scan_try_promote_parallel_index_scan` which on failure falls back cleanly to `S_INDX_SCAN`.

## Related

- Parent (template manager): [[components/parallel-heap-scan|parallel-heap-scan]] (will be renamed to `parallel-scan` on PR #7062 merge)
- Sibling variants: [[components/parallel-list-scan|parallel-list-scan]] (list), [[components/parallel-heap-scan-input-handler|parallel-heap-scan-input-handler]] (heap)
- Source dump: [[sources/2026-04-29-cbrd-26722-parallel-index-on-partitioned-tables|CBRD-26722 knowledge dump]]
- Underlying machinery: [[components/scan-manager|scan-manager]] § "SCAN_ID: The Polymorphic Scan Handle", [[components/btree|btree]], [[components/xasl|xasl]] § "Serialisation protocol"
- Partition runtime context: [[components/partition-pruning|partition-pruning]]
- Tracking PR: [[prs/PR-7062-parallel-scan-all-types|PR #7062]]
- Jira: CBRD-26722
