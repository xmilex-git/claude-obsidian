---
type: pr
pr_number: 6689
pr_url: "https://github.com/CUBRID/cubrid/pull/6689"
repo: "CUBRID/cubrid"
state: MERGED
is_draft: false
author: "xmilex-git"
merged_at: "2025-12-11T06:46:04Z"
closed_at: "2025-12-11T06:46:04Z"
merge_commit: "dedd387e6623914f498bd3afa024b404901ff1ca"
base_ref: "feature/refactor-pgbuf"
head_ref: "cas_pgbuf_2"
base_sha: "8b69f50d70c9a15473a68eba59cf80e85d840347"
head_sha: "0452cdae3985c49f443f213757ad4e1e3aec5e60"
jira: "CBRD-26425"
files_changed:
  - "src/connection/server_support.c"
  - "src/query/vacuum.c"
  - "src/session/session.c"
  - "src/storage/page_buffer.c"
  - "src/storage/page_buffer.h"
  - "src/thread/thread_entry.cpp"
  - "src/thread/thread_entry.hpp"
  - "src/thread/thread_entry_task.cpp"
related_components:
  - "[[components/page-buffer]]"
  - "[[components/thread-manager]]"
  - "[[components/storage]]"
  - "[[components/vacuum]]"
  - "[[components/session]]"
  - "[[components/connection]]"
related_sources:
  - "[[sources/cubrid-src-storage]]"
  - "[[sources/cubrid-src-thread]]"
ingest_case: b
triggered_baseline_bump: false
baseline_before: "175442fc858bd0075165729756745be6f8928036"
baseline_after: "175442fc858bd0075165729756745be6f8928036"
reconciliation_applied: false
reconciliation_applied_at:
incidental_enhancements_count: 2
tags:
  - pr
  - cubrid
  - storage
  - page-buffer
  - concurrency
  - atomic
  - lockfree
  - refactor
  - merged
created: 2026-04-24
updated: 2026-04-24
status: merged
---

# PR #6689 — Replace BCB mutex lock with atomic latch

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `MERGED` · **Author:** `@xmilex-git` · **Jira:** [CBRD-26425](https://jira.cubrid.org/browse/CBRD-26425)
> **Base → Head:** `feature/refactor-pgbuf` (`8b69f50d`) → `cas_pgbuf_2` (`0452cdae`)
> **Merge commit:** `dedd387e` · **Merged:** 2025-12-11 06:46:04 UTC
> **Scale:** 8 files, +950 / −502 LOC (1452 total). Largest file `page_buffer.c` at +932/−497 — Scale rule triggered, analyzed via 2 parallel deep-read subagents.
> **Review discussion:** none — zero reviews, zero inline comments, zero authoritative threads. Only `/run all` CI triggers by the author. Entire analysis is code-only.

> [!note] Ingest classification: case (b) — absorbed via sibling re-merge
> By strict hash classification this PR is **case (d)** — its merge commit `dedd387e` is NOT an ancestor of the wiki baseline `175442fc`. PR #6689 merged into a long-lived feature branch `feature/refactor-pgbuf`, not into `develop`. However, the equivalent changes **are** present in the current baseline via a **sibling re-merge PR #6704** (commit `58cef8e01` titled identically — `[CBRD-26425] Replace bcb mutex lock into atomic_latch`). The baseline `src/storage/page_buffer.c` contains 83 `atomic_latch` references and declares `pgbuf_thread_variables_init` at `page_buffer.h:481`, confirming the feature is live. Treating as retroactive doc (case b in substance) — no Reconciliation Plan, no baseline bump. Drift between `dedd387e` (what the PR actually changed) and the baseline (what PR #6704 ultimately landed) is flagged in the "Baseline drift vs PR head" section below.

## Summary

Replaces the per-BCB (Buffer Control Block) `pthread_mutex_t` with a 64-bit `std::atomic<uint64_t>` whose bits pack `{latch_mode: uint16_t, waiter_exists: uint16_t, fcnt: int32_t}`. All single-owner state transitions now fold into a `compare_exchange_weak` on this word, eliminating thousands of mutex lock/unlock pairs per second on the hot page-fix path. Adds a lockfree read-only fast path (`pgbuf_lockfree_fix_ro` + `pgbuf_search_hash_chain_no_bcb_lock` + `pgbuf_lockfree_unfix_ro`) that bypasses both the hash-anchor mutex and the BCB mutex when the caller only needs a READ latch on a resident, non-transitioning page. Hoists two per-thread fields (`m_is_private_lru_enabled` bool + `m_holder_anchor` pointer cache) onto `cubthread::entry` and adds `pgbuf_thread_variables_init` called from 3 thread-reassignment sites (CSS worker dispatch, vacuum master wake-up, session attach).

## Motivation

The BCB mutex was the single biggest source of contention on the page-fix path under concurrent OLTP load — a single hot page (e.g. a catalog root B-tree page, a heap header, an index root) would serialise every reader through one `pthread_mutex_t`. The atomic-latch design collapses `{read-lock / modify-fcnt / release}` into a single CAS on a 64-bit word, and the lockfree RO path skips even that CAS in the common "page already present, no state transition" case. Jira CBRD-26425 targets pgbuf throughput for workloads dominated by read-heavy index scans and catalog probes.

## Baseline drift vs PR head

Because the ingested PR landed into `feature/refactor-pgbuf` while the baseline absorbed the changes through sibling PR #6704, the final baseline code differs slightly from this PR's head commit. Three drifts identified by deep-read:

1. **`pgbuf_lockfree_fix_ro` VPID recheck inside CAS gate** (baseline `page_buffer.c:7465-7477`) — added by PR #6704, not present in `0452cdae`. Defends against ABA where a BCB is recycled to a different VPID between the `pgbuf_search_hash_chain_no_bcb_lock` lookup and the CAS.
2. **`OLD_PAGE_MAYBE_DEALLOCATED` accepted as a valid `fetch_mode`** in lockfree fix (baseline `page_buffer.c:2136, 7454`) — widens the RO fast path to more caller sites.
3. **`latch_last_thread` widened from `SERVER_MODE && !NDEBUG` to unconditional `SERVER_MODE`** (baseline `page_buffer.c:519`). The diff header and comment still say `!NDEBUG` — stale documentation.

These are attributable to PR #6704, not PR #6689. When updating wiki component pages, cite #6704's merge commit for these three items.

## Changes

### Structural

**New type** in `page_buffer.c` (internal):
```c
typedef std::atomic<uint64_t> PGBUF_ATOMIC_LATCH;

union pgbuf_atomic_latch_impl {
  uint64_t raw;
  struct {
    PGBUF_LATCH_MODE latch_mode;  // enum, 16 bits via underlying uint16_t
    uint16_t         waiter_exists;
    int              fcnt;        // 32-bit signed fix-count
  } impl;
};
typedef union pgbuf_atomic_latch_impl PGBUF_ATOMIC_LATCH_IMPL;
```

`PGBUF_LATCH_MODE` (in `page_buffer.h:190-197`) is an `enum : uint16_t` so it fits the 16-bit slot. Total = 16 + 16 + 32 = 64 bits; `std::atomic<uint64_t>` is lock-free on every supported platform.

**BCB struct change** (`page_buffer.c` around line 494):
- Removed: `int fcnt`, `PGBUF_LATCH_MODE latch_mode`.
- Added: `PGBUF_ATOMIC_LATCH atomic_latch`.
- Added under `SERVER_MODE` (not `!NDEBUG` in baseline — see drift): `THREAD_ENTRY *latch_last_thread`.
- Retained: `int owner_mutex` (existing debug field).

**Counters promoted to `std::atomic_int`** (`page_buffer.c:697, 699`):
- `pg_unfix_cnt` — previously plain `int`, now atomic for quota-adjust refresh.
- `fix_req_cnt` — ditto.

**New public API** (`page_buffer.h:481`):
```c
void pgbuf_thread_variables_init (THREAD_ENTRY *thread_p);
```

**New `cubthread::entry` public fields** (`thread_entry.hpp:311-312`):
```cpp
bool m_is_private_lru_enabled;
struct pgbuf_holder_anchor *m_holder_anchor;
```
Forward declared at `thread_entry.hpp:57`. Both default to `false` / `NULL` in the constructor (`thread_entry.cpp:142-143`).

**Removed function**: `pgbuf_latch_idle_page` — inlined into `pgbuf_latch_bcb_upon_fix` (`page_buffer.c:6121-6129`).

**New functions** in `page_buffer.c`:
- CAS primitives: `set_latch`, `add_fcnt`, `set_latch_and_fcnt`, `set_latch_and_add_fcnt`, `set_waiter_exists`, `get_fcnt`, `get_waiter_exists`, `get_latch`, `get_impl` (all `STATIC_INLINE`, at `page_buffer.c:1310-1412`).
- `pgbuf_thread_variables_init` (`page_buffer.c:1415-1433`).
- Lockfree RO path: `pgbuf_lockfree_fix_ro` (`:7451-7514`), `pgbuf_search_hash_chain_no_bcb_lock` (`:7516-7531`), `pgbuf_lockfree_unfix_ro` (`:7533-7556`).

### Per-file notes

- `src/storage/page_buffer.c` (+932/−497) — core refactor: BCB field change, CAS primitives, lockfree RO fix/unfix, rewrite of `pgbuf_latch_bcb_upon_fix` / `pgbuf_unlatch_bcb_upon_unfix` / `pgbuf_promote_read_latch_release` as CAS loops, `PGBUF_THREAD_HAS_PRIVATE_LRU` now reads the cached bool ([[components/page-buffer]]).
- `src/storage/page_buffer.h` (+6/−5) — public prototype for `pgbuf_thread_variables_init`, `PGBUF_LATCH_MODE` enum underlying-type fixed to `uint16_t` ([[components/page-buffer]]).
- `src/thread/thread_entry.hpp` (+5) — new public fields + forward decl ([[components/thread-manager]]).
- `src/thread/thread_entry.cpp` (+2) — constructor init-list entries ([[components/thread-manager]]).
- `src/thread/thread_entry_task.cpp` (+1) — `retire_context` resets `m_is_private_lru_enabled = false` (pairs with existing `private_lru_index = -1` reset). `recycle_context` does **not** reset the bool — asymmetry, see smell #3 below ([[components/thread-manager]]).
- `src/query/vacuum.c` (+2) — `pgbuf_thread_variables_init` call at `vacuum_master_task::execute:3010` ([[components/vacuum]]).
- `src/session/session.c` (+1) — call at `session_set_conn_entry_data:2791`, right after `private_lru_index` assignment ([[components/session]]).
- `src/connection/server_support.c` (+1) — call at `css_server_task::execute:2766`, right after `private_lru_index = session_get_private_lru_idx(session_p)` ([[components/connection]]).

### Behavioral

1. **Packed-word atomicity.** All four state-mutating CAS primitives (`set_latch`, `add_fcnt`, `set_latch_and_fcnt`, `set_latch_and_add_fcnt`, `set_waiter_exists`) use the same pattern:
   ```cpp
   do {
     impl.raw = latch->load (memory_order_acquire);
     new_impl = impl;  // copy-then-mutate
     new_impl.impl.<field> = <value>;
   } while (!latch->compare_exchange_weak (impl.raw, new_impl.raw,
                                           memory_order_acq_rel,
                                           memory_order_acquire));
   ```
   `compare_exchange_weak` is chosen for the spurious-failure retry loop on relaxed ISAs (ARM64). `_and_` compound setters (e.g. `set_latch_and_fcnt`) atomically update both fields in a single CAS — a reader can never observe a half-updated state where `latch_mode` changed but `fcnt` didn't.
2. **Lockfree RO fast path.** `pgbuf_lockfree_fix_ro` (entered from `pgbuf_fix` when the caller wants READ latch on `OLD_PAGE` or `OLD_PAGE_MAYBE_DEALLOCATED` and the page is already resident in the hash chain) performs:
   - Hash chain traversal **without** the anchor mutex (`pgbuf_search_hash_chain_no_bcb_lock`).
   - Gate CAS that (a) verifies VPID still matches (ABA defense added by PR #6704), (b) verifies `latch_mode ∈ {NO_LATCH, READ}`, (c) verifies `waiter_exists == false`, (d) atomically increments `fcnt`.
   - On success: returns `PAGE_PTR` without ever touching the BCB mutex.
   - On failure: falls back to `pgbuf_fix` slow path which takes hash mutex → BCB CAS loop.
3. **Lockfree unfix counterpart.** `pgbuf_lockfree_unfix_ro` (entered from `pgbuf_unfix:2965`) does the symmetric CAS: decrement `fcnt`, and only if the resulting `fcnt == 0 && waiter_exists == false` return without waking. If `waiter_exists` is true, the slow path takes over to dequeue and wake a waiter under mutex. This is safe because the slow path is idempotent under `waiter_exists = true`.
4. **`waiter_exists` semantics.** Set by blocking threads immediately before `thread_lock_entry`; cleared by the holder on the last unfix that empties the waiter list. Has the classic lost-wakeup risk in any waiter-count scheme — the CAS primitive guarantees the bit is consistent with the queue state at each transition, and the `thread_lock_entry` / `thread_unlock_entry` mutex provides the final barrier against missed wakeups. See smell #1 for an identified race in `pgbuf_wakeup_reader_writer`.
5. **Latch promotion** (`pgbuf_promote_read_latch_release:2624-2835`). The `PGBUF_PROMOTE_ONLY_READER` path is now a single CAS: `{latch_mode=READ, fcnt=1, waiter_exists=false}` → `{latch_mode=WRITE, fcnt=1, waiter_exists=*}`. `PGBUF_PROMOTE_SHARED_READER` still blocks on readers but uses `set_waiter_exists(true)` + `thread_suspend_*` instead of mutex wait-queues internally.
6. **`pgbuf_latch_idle_page` semantics absorbed.** The old helper ran a non-CAS "I'm the first latcher on a clean idle page" fast path; it is now a CAS branch inside `pgbuf_latch_bcb_upon_fix:6121-6129` that handles `latch_mode == NO_LATCH` in the same retry loop as contended latching.
7. **`PGBUF_THREAD_HAS_PRIVATE_LRU` redefinition.** Was: `PGBUF_PAGE_QUOTA_IS_ENABLED && thread_p != NULL && thread_p->private_lru_index != -1`. Now: `thread_p != NULL && thread_p->m_is_private_lru_enabled` (`page_buffer.c:958-961`). The cached bool is populated by `pgbuf_thread_variables_init` from the same underlying predicate. Every `pgbuf_fix*` call now checks one bool on the owning thread's `entry` rather than three loads (quota, thread, index). Net: 2 fewer loads per fix on the hot path, but introduces the invariant that `private_lru_index` writers must call `pgbuf_thread_variables_init` after every write (see smell #3).
8. **`m_holder_anchor` caching.** The pre-PR `thrd_holder_info = &pgbuf_Pool.thrd_holder_info[thread_p->index]` array index is now a cached pointer on the thread entry. Lazily bound at four holder-accessor sites (`page_buffer.c:5799, 5882, 5993, 13063`) so daemons that skip the three init sites still get a valid pointer on first use. Cleared in bulk by `thread_clear_all_holder_anchor` (called from `pgbuf_finalize:1981`) before the underlying `thrd_holder_info` array is freed — dangling-pointer hazard if shutdown order ever inverts.
9. **Atomic counter promotion.** `pg_unfix_cnt` and `fix_req_cnt` were plain `int` previously — any concurrent increment lost updates and, under aggressive TSAN, would trip races. Promoting to `std::atomic_int` fixes both correctness and visibility.
10. **Init sites for `pgbuf_thread_variables_init`.** Three sites, all immediately after a `private_lru_index` write:
    - `css_server_task::execute` (`server_support.c:2766`) — every css worker task dispatch that has an attached session.
    - `vacuum_master_task::execute` (`vacuum.c:3010`) — vacuum master loop entry after the boot/shutdown guards.
    - `session_set_conn_entry_data` (`session.c:2791`) — session attach / reconnect path, mid-task.
    `entry_manager::retire_context` (`thread_entry_task.cpp:76`) resets `m_is_private_lru_enabled = false` for pooled entries, mirroring the `private_lru_index = -1` reset.

### New surface (no existing wiki reference)

- `PGBUF_ATOMIC_LATCH` type and the `{latch_mode, waiter_exists, fcnt}` packed union — not documented on [[components/page-buffer]] (which still lists `mutex: pthread_mutex_t` as a BCB field). Incidental enhancement applied below.
- `pgbuf_lockfree_fix_ro` / `pgbuf_search_hash_chain_no_bcb_lock` / `pgbuf_lockfree_unfix_ro` — entirely new subsystem not in the wiki.
- `pgbuf_thread_variables_init` + the cached `m_is_private_lru_enabled` / `m_holder_anchor` fields — not in [[components/thread-manager]] (which does not yet exist as a detailed page). Noted as a follow-up.

## Review discussion highlights

**None.** Zero reviews, zero inline comments, zero authoritative design threads. `gh api repos/CUBRID/cubrid/pulls/6689/comments` returned an empty array. Only `/run all` CI-trigger comments from the author exist — all non-authoritative.

The commit log itself is the only narrative signal. Timeline of commit headlines (chronological):
- `make all cas` — initial sweep replacing mutex with CAS loops.
- `ro fix complete` — lockfree RO fast path lands.
- `lockfree unfix` — symmetric lockfree unfix.
- `enhance holder` + `Revert "enhance holder"` — a holder-reclamation experiment that was rolled back.
- `race condition 제거` — a race fix (likely the CAS retry loop stabilisation).
- `latch promote bug fix` — promotion path correctness fix.
- `remove latch idle page` — `pgbuf_latch_idle_page` helper absorbed into the CAS loop.
- `handle else` — coverage for a previously-fallthrough branch.
- `private lru index fix` — the `m_is_private_lru_enabled` caching bug.
- `debugging info : latch_last_thread` — debug field added.
- `no core` — final pass to stop a crash path.

No reviewer sign-offs. The PR merged directly into the feature branch under the author's own approval.

## Reconciliation Plan

n/a — PR's changes are already absorbed in the current baseline via sibling PR #6704. No deferred reconciliation needed.

## Pages Reconciled

n/a (case b).

## Incidental wiki enhancements

Applied during this ingest. These are baseline-only facts — they stand regardless of which merge path the changes took.

1. **[[components/page-buffer]]** — updated the BCB struct table: replaced the stale `mutex | pthread_mutex_t | Protects BCB state transitions` row with the new `atomic_latch | PGBUF_ATOMIC_LATCH (std::atomic<uint64_t>) | Packed {latch_mode: uint16_t, waiter_exists: uint16_t, fcnt: int32_t} — single-word CAS replaces the old mutex for all state transitions`. Also added a new "Atomic latch model" section near the end describing the CAS primitives, the lockfree RO fast path (`pgbuf_lockfree_fix_ro` + `pgbuf_search_hash_chain_no_bcb_lock` + `pgbuf_lockfree_unfix_ro`), and the `pgbuf_thread_variables_init` contract with `private_lru_index` writers.
2. **[[components/page-buffer]]** — added `pgbuf_thread_variables_init` to the public-API list in the frontmatter + body, and noted the `PGBUF_THREAD_HAS_PRIVATE_LRU` change from three-load predicate to one-bool cached read.

## Deep analysis — supplementary findings

Synthesized from two subagent reports. All are code-only observations (no reviewer discussion exists to validate). Several warrant a follow-up PR.

### Correctness — latent bugs

1. **`pgbuf_wakeup_reader_writer` potential double-lock on CAS retry.** `page_buffer.c:7249` — inside the CAS retry loop, `thread_lock_entry` is called each iteration. If the CAS fails spuriously and retries, the same `thrd_entry` gets locked twice without an intervening unlock on the first iteration's path. Real correctness bug — warrants a follow-up PR to move the lock acquisition outside the CAS loop or ensure per-iteration unlock. (Contrast: `pgbuf_wakeup_reader_writer` at line 6980 correctly places `thread_lock_entry` after `if (can_grant)`, outside the CAS.)
2. **`pgbuf_unlatch_bcb_upon_unfix` fcnt<0 recovery never CASes the clamp.** `page_buffer.c:6449-6463` — on detecting `fcnt < 0` after decrement the code logs and asserts but does not atomically rewrite the latch to clamp fcnt back to 0. The atomic stays corrupt. In release builds the assert is elided; the next reader sees `fcnt == -N` and either (a) underflows further on decrement, or (b) fails `fcnt == 0` guards at subsequent latch releases.
3. **`pgbuf_bcb_register_fix` bypassed on lockfree fix path.** The hot-page detection / LRU-boost accounting done by `pgbuf_bcb_register_fix` is called only from the slow path — every lockfree RO fix silently skips registration. Hot-page detection is now skewed towards pages that miss the fast path (i.e., contended or transitioning pages), reversing the signal.
4. **`pgbuf_assign_private_lru` / `pgbuf_release_private_lru` may not re-run `pgbuf_thread_variables_init`.** These mutate `private_lru_index` outside the three blessed sites. If they don't call init after the mutation, `m_is_private_lru_enabled` goes stale and the `PGBUF_THREAD_HAS_PRIVATE_LRU` fast-path predicate returns the wrong answer. Needs a code check to confirm (the subagent flagged as "needs verification"); if confirmed, pages fixed between the index change and the next init call route to the wrong LRU. Subtle but meaningful correctness impact.
5. **`recycle_context` asymmetry.** `thread_entry_task.cpp:95` resets `private_lru_index = -1` but does **not** reset `m_is_private_lru_enabled`. Safe today because `retire_context` always precedes `recycle_context` and resets both — but a future reorder would leak stale-true into a recycled entry.

### Correctness — subtler

6. **Lazy-init of `m_holder_anchor` at four sites, no documented contract.** Daemons (vacuum workers, log flush, checkpoint, DWB, CDC) never call `pgbuf_thread_variables_init`. The four holder-accessor sites (`page_buffer.c:5799, 5882, 5993, 13063`) all have `if (!thread_p->m_holder_anchor) thread_p->m_holder_anchor = &pgbuf_Pool.thrd_holder_info[thread_p->index];` guards. If a future holder-accessor is added without the guard, daemons crash on NULL deref. Needs a policy comment or a wrapper accessor.
7. **`thread_clear_all_holder_anchor` called from exactly one site.** `page_buffer.c:1981` (inside `pgbuf_finalize`). If a second lifecycle path ever needs to tear down pgbuf (dynamic re-init, shrink_thread_pool), entries hold dangling pointers. Invariant not documented.
8. **No NULL-guard on `pgbuf_Pool.thrd_holder_info` in `pgbuf_thread_variables_init`.** Post-finalize pre-reinit window is unprotected. Reachable only by unusual shutdown sequences but the defence is thin.
9. **ABA in the lockfree RO path.** The PR #6689 version did not recheck VPID inside the gate CAS — purely relying on `waiter_exists` and `latch_mode` bits. PR #6704 added the VPID recheck at `page_buffer.c:7465-7477`. Without that recheck the fast path could return a `PAGE_PTR` for a BCB whose VPID was just recycled. Baseline has the fix; the original PR #6689 head did not.
10. **`pgbuf_simple_fix` / `pgbuf_simple_unfix` still take BCB mutex.** Around pure atomic operations (`page_buffer.c:2576, 2577, 2611`). Either the mutex is vestigial (can be removed) or it protects state that the atomic-latch design forgot about. Dead-weight if the former, correctness gap if the latter.

### Code concerns / smells

11. **`copy_bcb` dead code.** `page_buffer.c:1282-1308`. Never called anywhere in baseline. Remove or gate with `#if 0`.
12. **Stale comment at `page_buffer.c:520`.** Claims `latch_last_thread` is gated by `SERVER_MODE && !NDEBUG`; actual baseline gating is `SERVER_MODE` only (widened by PR #6704). Fix: update the comment or re-narrow the gate.
13. **Three disparate init sites for `pgbuf_thread_variables_init`** is a fragile design. A future site that writes `private_lru_index` without calling init silently breaks the cache. An encapsulated setter on `cubthread::entry` would make the invariant enforceable.
14. **Public `m_is_private_lru_enabled` and `m_holder_anchor` fields.** Matches the file's pattern of public-for-legacy but no documented contract prevents a cross-thread writer. In practice single-owner, but worth a comment.
15. **`pgbuf_thread_variables_init` is not server-mode-gated.** SA_MODE falls through to an always-`false` branch (via `pgbuf_Pool.quota.num_private_LRU_list == 0` gate). Works, but the function name implies SERVER_MODE and SA_MODE callers are a code-path audit hazard.

### Performance

16. **`PGBUF_THREAD_HAS_PRIVATE_LRU` is on the hot path.** The PR's optimization of caching the predicate into one bool (saving 2 loads per fix) is an implicit signal that this gate is called ≥ millions of times per second on the profile. Worth documenting on the wiki's page-buffer page.
17. **`pg_unfix_cnt` / `fix_req_cnt` cache-line contention.** Promoted to `std::atomic_int`; without alignment padding, they sit on the same cache line as neighbouring monitor counters and will see fetch_add ping-pong across cores. Not flagged in diff — worth a memory-layout review.

## Baseline impact

- Before: `175442fc858bd0075165729756745be6f8928036`
- After: `175442fc858bd0075165729756745be6f8928036` (unchanged — changes already in baseline)
- Bump triggered: `false`
- Rationale: case (b) in substance — feature branch merge; the absorbing sibling PR #6704 landed the same content into `develop` before the wiki baseline was taken. Logged: see [[log]] entry `[2026-04-24] pr-ingest PR #6689`.

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/6689
- Sibling re-merge PR: [CBRD-26425] Replace bcb mutex lock into atomic_latch (#6704) — commit `58cef8e01`. Not separately ingested; its changes == this PR's changes plus three minor drifts (VPID recheck, `OLD_PAGE_MAYBE_DEALLOCATED` support, `latch_last_thread` gating).
- Jira: [CBRD-26425](https://jira.cubrid.org/browse/CBRD-26425)
- Components touched: [[components/page-buffer]], [[components/thread-manager]], [[components/storage]], [[components/vacuum]], [[components/session]], [[components/connection]]
- Sources: [[sources/cubrid-src-storage]], [[sources/cubrid-src-thread]]
- Adjacent PRs: [[prs/PR-6911-parallel-heap-scan-io-bottleneck]] (later refactor on the same `page_buffer.c` hot path), [[prs/PR-7062-parallel-scan-all-types]] (uses the lockfree RO fix in its heap scan loop)
