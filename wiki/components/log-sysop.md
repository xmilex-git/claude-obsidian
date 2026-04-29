---
type: component
address: c-000005
parent_module: "[[components/transaction|transaction]]"
parent_component: "[[components/log-manager|log-manager]]"
path: "src/transaction/log_manager.c, log_record.hpp, log_manager.h"
status: active
purpose: "System Operation logging — nested transaction-within-a-transaction primitive; the `log_sysop_*()` family of public APIs for catalog updates, B-tree mutations, and other atomic-or-rollback fragments inside a larger user transaction"
key_files:
  - "src/transaction/log_manager.c (definitions, lines 3563-4178 + log_sysop_do_postpone at 8189)"
  - "src/transaction/log_manager.h:200-215 (public extern declarations)"
  - "src/transaction/log_record.hpp:67 (LOG_SYSOP_END rectype), :285-301 (LOG_SYSOP_END_TYPE enum), :304-324 (LOG_REC_SYSOP_END struct), :327-332 (LOG_REC_SYSOP_START_POSTPONE struct)"
  - "src/transaction/log_tran_table.h (LOG_TDES.topops stack, on_sysop_start / on_sysop_end hooks, lock_topop / unlock_topop)"
public_api:
  - "log_sysop_start(thread_p) — push a new sysop frame on TDES.topops"
  - "log_sysop_start_atomic(thread_p) — same, plus emit LOG_SYSOP_ATOMIC_START marker"
  - "log_sysop_commit(thread_p) — emit LOG_SYSOP_END_COMMIT and pop"
  - "log_sysop_abort(thread_p) — rollback then emit LOG_SYSOP_END_ABORT and pop"
  - "log_sysop_attach_to_outer(thread_p) — fold this sysop into its parent (no log record)"
  - "log_sysop_end_logical_undo(thread_p, rcvindex, vfid, undo_size, undo_data) — commit + carry logical-undo payload"
  - "log_sysop_end_logical_compensate(thread_p, undo_nxlsa) — commit + carry compensate LSA"
  - "log_sysop_end_logical_run_postpone(thread_p, posp_lsa) — commit + carry run-postpone LSA"
  - "log_sysop_end_recovery_postpone(thread_p, log_record, data_size, data) — recovery-time variant of commit"
  - "log_sysop_end_type_string(end_type) → const char* (debug)"
  - "log_check_system_op_is_started(thread_p) → bool"
tags:
  - component
  - cubrid
  - logging
  - transaction
  - recovery
  - sysop
  - wal
related:
  - "[[components/log-manager|log-manager]]"
  - "[[components/transaction|transaction]]"
  - "[[components/recovery|recovery]]"
  - "[[components/vacuum|vacuum]]"
  - "[[components/btree|btree]]"
  - "[[components/heap-file|heap-file]]"
created: 2026-04-29
updated: 2026-04-29
---

# Log Sysop — System Operation Logging Family

**System operation** (sysop) = a *nested transaction-within-a-transaction* mechanism. While a user transaction commits or aborts as a single atomic unit at end-of-statement-block, individual subsystems often need a **smaller atomic unit** that commits independently of the surrounding transaction's outcome:

- a B-tree page split that must succeed-or-fail as a whole, regardless of what happens to the SQL statement that triggered it;
- a system catalog update (e.g. statistics refresh) that must persist even if the user later rolls back;
- a logical-level operation (e.g. heap relocation) whose physical undo cannot be expressed in a single log record.

The sysop primitive lets these fragments **commit independently** (their changes become permanent, regardless of the parent transaction's fate) **or abort independently** (rollback to the sysop's start LSA without aborting the parent).

The implementation is a public C API exposed by [[components/log-manager|log-manager]] under the `log_sysop_*` prefix in `src/transaction/log_manager.c:3557-4178`. The bulk of the family is 18 functions; only 9 are exposed to other subsystems (`log_manager.h:200-215`), the other 9 are static helpers.

## Conceptual model

```
User TX begins
  │
  ├─ log_sysop_start ─────────── sysop level 0 starts
  │     │
  │     ├─ log_append_undoredo_data (regular log record A)
  │     ├─ log_append_undoredo_data (regular log record B)
  │     │
  │     ├─ log_sysop_start ───── sysop level 1 starts (NESTED)
  │     │     │
  │     │     ├─ log_append_undoredo_data (regular log record C)
  │     │     │
  │     │     └─ log_sysop_commit  ─ pop level 1, emit LOG_SYSOP_END
  │     │
  │     └─ log_sysop_commit  ──── pop level 0, emit LOG_SYSOP_END
  │
  └─ log_commit  ─── user TX commits (nothing more)
```

Sysops form a **stack** on the `LOG_TDES.topops` array. `tdes->topops.last` is the current depth (-1 = no sysop active). `tdes->topop_lsa` always reflects the start-LSA of the innermost active sysop, used as the rollback target for aborts.

## The 18-function family

Function definitions live in `src/transaction/log_manager.c` between line 3563 (`log_sysop_end_type_string`) and line 4178 (`log_sysop_get_tran_index_and_tdes`), with `log_sysop_do_postpone` at line 8189.

### Public lifecycle entry points (exposed via `log_manager.h`)

| Function | File:line | Role |
|---|---|---|
| `log_sysop_start` | `log_manager.c:3599` | Push a fresh sysop frame onto `tdes->topops`. Captures `tdes->tail_lsa` as the frame's `lastparent_lsa`. **No log record emitted at start** — only at end. |
| `log_sysop_start_atomic` | `log_manager.c:3665` | Calls `log_sysop_start`, then emits `LOG_SYSOP_ATOMIC_START` (record type 50). Marks the sysop as "must rollback during recovery if encountered incomplete." See *Atomic sysops* below. |
| `log_sysop_commit` | `log_manager.c:3916` | Emits `LOG_SYSOP_END` with `type=LOG_SYSOP_END_COMMIT` and pops the frame. The default success path. |
| `log_sysop_abort` | `log_manager.c:4038` | Rolls back to the frame's `lastparent_lsa` via `log_rollback`, then emits `LOG_SYSOP_END` with `type=LOG_SYSOP_END_ABORT`, then pops. |
| `log_sysop_attach_to_outer` | `log_manager.c:4097` | Pop the frame **without** emitting any log record — the sysop's changes get re-attributed to the immediate parent (sysop or transaction). Used when a sysop turned out to be unnecessary as a separate unit. Transfers any pending postpone LSA up to the parent. |
| `log_sysop_end_logical_undo` | `log_manager.c:3941` | Commit + emit `LOG_SYSOP_END` with `type=LOG_SYSOP_END_LOGICAL_UNDO` (or `_LOGICAL_MVCC_UNDO` if `LOG_IS_MVCC_OPERATION(rcvindex)`). The end record carries logical-undo payload that recovery/rollback will replay through `RV_fun[rcvindex].undofun`. |
| `log_sysop_end_logical_compensate` | `log_manager.c:3984` | Commit + emit `LOG_SYSOP_END_LOGICAL_COMPENSATE`. Used when this sysop's purpose was to **undo** an earlier change — the end record carries `undo_nxlsa` (the LSA the undo cursor should jump to next). |
| `log_sysop_end_logical_run_postpone` | `log_manager.c:4003` | Commit + emit `LOG_SYSOP_END_LOGICAL_RUN_POSTPONE`. Used when this sysop *executed* a postpone record — the end carries the postpone LSA that was just satisfied. |
| `log_sysop_end_recovery_postpone` | `log_manager.c:4024` | Recovery-time wrapper — replays a `_LOGICAL_RUN_POSTPONE` end record during crash recovery. Sets `is_rv_finish_postpone=true` on the internal helper. |

### Static helpers (not exposed)

| Function | File:line | Role |
|---|---|---|
| `log_sysop_end_type_string` | `log_manager.c:3570` | Debug printer: maps `LOG_SYSOP_END_TYPE` enum to its identifier string. Public via header. |
| `log_sysop_end_random_exit` | `log_manager.c:3707` | **Crash injector**. With `FI_TEST_LOG_MANAGER_RANDOM_EXIT_AT_END_SYSTEMOP` enabled (via `--fault-injection` in test builds), randomly aborts the process with 1/5000 probability inside end-functions. Used by recovery test harnesses. |
| `log_sysop_end_begin` | `log_manager.c:3724` | Common preamble of end-functions: invokes the random-exit hook, fetches `tran_index` + `tdes`, asserts `tdes->topops.last >= 0`. |
| `log_sysop_end_unstack` | `log_manager.c:3746` | Decrement `tdes->topops.last`; restore `tdes->topop_lsa` to the new innermost frame's `lastparent_lsa` (or `NULL_LSA` if stack now empty). |
| `log_sysop_end_final` | `log_manager.c:3767` | Common postamble: unstack, unlock the topop mutex, increment `PSTAT_TRAN_NUM_END_TOPOPS`, vacuum-state log, **trigger checkpoint** if `LOG_ISCHECKPOINT_TIME()` and we just ended the outermost sysop. |
| `log_sysop_commit_internal` | `log_manager.c:3825` | The shared body of all commit-style end-functions. Decides whether the sysop is "empty" (no log records appended since `log_sysop_start`) and short-circuits, otherwise: validates `tdes->state` against the requested `LOG_SYSOP_END_TYPE`, calls `log_sysop_do_postpone`, appends `LOG_SYSOP_END`, updates `tail_topresult_lsa`. |
| `log_sysop_get_level` | `log_manager.c:4145` | Returns `tdes->topops.last` — current nesting depth, or -1 if none. |
| `log_sysop_get_tran_index_and_tdes` | `log_manager.c:4169` | Resolves `(tran_index, tdes)` from the thread entry. Centralises the lookup so vacuum's special tdes-redirection rule lives in one place. |
| `log_sysop_do_postpone` | `log_manager.c:8189` | If the sysop accumulated postpone records (`tdes->topops.stack[last].posp_lsa != NULL_LSA`), append `LOG_SYSOP_START_POSTPONE` (rectype 18), then run the postpone records from the postpone-cache (fast path) or by forward-scan via `log_do_postpone`. Restores `tdes->state` to its value before the postpone phase. |

### Convenience query

| Function | File:line | Role |
|---|---|---|
| `log_check_system_op_is_started` | `log_manager.c:4187` | Predicate — is the calling thread currently inside any sysop? Used by assertions throughout the engine. |

## Six end-record subtypes

`LOG_SYSOP_END` is record type 20, but its body's `type` field selects one of six variants. Defined in `log_record.hpp:285-301`:

```c
enum log_sysop_end_type {
  LOG_SYSOP_END_COMMIT,                /* permanent changes */
  LOG_SYSOP_END_ABORT,                 /* aborted system op */
  LOG_SYSOP_END_LOGICAL_UNDO,          /* logical undo */
  LOG_SYSOP_END_LOGICAL_MVCC_UNDO,     /* logical mvcc undo */
  LOG_SYSOP_END_LOGICAL_COMPENSATE,    /* logical compensate */
  LOG_SYSOP_END_LOGICAL_RUN_POSTPONE   /* logical run postpone */
};
```

The end-record (`LOG_REC_SYSOP_END`, `log_record.hpp:304-324`) carries:

```c
struct log_rec_sysop_end {
  LOG_LSA lastparent_lsa;        /* parent's tail_lsa at sysop_start */
  LOG_LSA prv_topresult_lsa;     /* last sysop end (commit or abort) */
  LOG_SYSOP_END_TYPE type;
  const VFID *vfid;              /* file-id, used to derive TDE info; same as
                                    mvcc_undo->vacuum_info if MVCC-undo */
  union {
    LOG_REC_UNDO undo;             /* type == LOGICAL_UNDO       */
    LOG_REC_MVCC_UNDO mvcc_undo;   /* type == LOGICAL_MVCC_UNDO  */
    LOG_LSA compensate_lsa;        /* type == LOGICAL_COMPENSATE */
    struct {
      LOG_LSA postpone_lsa;
      bool is_sysop_postpone;
    } run_postpone;                /* type == LOGICAL_RUN_POSTPONE */
  };
};
```

The `lastparent_lsa` is the **rollback anchor**: undo-recovery, encountering a `LOG_SYSOP_END_ABORT`, jumps the cursor backward to that LSA and resumes scanning, effectively skipping the aborted sysop's records. For `LOG_SYSOP_END_COMMIT` it serves as the no-op marker that the inner records have already been logically condensed.

## Atomic sysops

`log_sysop_start_atomic` emits an extra `LOG_SYSOP_ATOMIC_START` record (rectype 50, `log_record.hpp:130`) immediately after pushing the frame. The presence of that marker without a corresponding end record is detected during recovery and triggers an **unconditional rollback** of the whole atomic sysop *before* the postpone phase begins (`log_recovery.c` analysis pass).

Used by:

- **B-tree page splits / merges** — must not leave the index in a half-split state across crashes.
- **File-table allocation** — file-header writes paired with extent-bitmap updates.
- Other multi-page structural mutations whose intermediate state is corrupt-by-itself.

The marker is suppressed for nested atomic sysops (the parent's marker already covers them) — see `log_manager.c:3689-3697`.

## Empty-sysop short-circuit

`log_sysop_commit_internal` (`log_manager.c:3841-3846`) detects sysops that appended **no log records** since their start, and short-circuits without emitting `LOG_SYSOP_END`:

```c
if ((LSA_ISNULL (&tdes->tail_lsa) || LSA_LE (&tdes->tail_lsa, LOG_TDES_LAST_SYSOP_PARENT_LSA (tdes)))
    && (log_record->type == LOG_SYSOP_END_COMMIT || log_No_logging))
  {
    /* No change. */
    assert (LSA_ISNULL (&LOG_TDES_LAST_SYSOP (tdes)->posp_lsa));
  }
```

This avoids polluting the log with no-op pairs of `LOG_SYSOP_START`/`LOG_SYSOP_END`.

The complementary assertion *forbids* logical-end variants from being empty — a logical-undo/compensate/run-postpone with no inner records would hide a bug. The known exceptions are documented in the surrounding comment (e.g. `RVPGBUF_FLUSH_PAGE`); they emit a dummy log record explicitly to satisfy the invariant.

## Postpone interaction

A sysop can accumulate **postpone records** (deferred actions to run after the sysop commits, like file-deletes-on-commit). On commit, `log_sysop_do_postpone` (`log_manager.c:8189`):

1. If `LOG_TDES_LAST_SYSOP_POSP_LSA(tdes)` is null → nothing to postpone, return.
2. Append `LOG_SYSOP_START_POSTPONE` (rectype 18) — marks the start of the postpone-execution phase. Carries the *would-be* `LOG_SYSOP_END` and the first postpone LSA.
3. Try the postpone-cache fast path (`tdes->m_log_postpone_cache.do_postpone`). Hits → increment `PSTAT_TRAN_NUM_TOPOP_PPCACHE_HITS` and return.
4. Miss → forward-scan the log via `log_do_postpone` from the first postpone LSA, executing each postpone record. Increment `PSTAT_TRAN_NUM_TOPOP_PPCACHE_MISS`.
5. Restore `tdes->state` to its pre-postpone value.

The `LOG_SYSOP_START_POSTPONE` record exists to give recovery a single LSA to anchor the postpone-replay phase, separate from the eventual `LOG_SYSOP_END`.

`log_sysop_end_recovery_postpone` is the recovery-time entry point that replays this — given the original `LOG_REC_SYSOP_END` payload (read from the start-postpone record), it calls the internal commit with `is_rv_finish_postpone=true`, suppressing replication-log emission.

## Vacuum integration

`log_sysop_get_tran_index_and_tdes` is the **only** function in the family that knows about vacuum:

```c
*tran_index_out = LOG_FIND_THREAD_TRAN_INDEX (thread_p);
*tdes_out = LOG_FIND_TDES (*tran_index_out);
```

`LOG_FIND_THREAD_TRAN_INDEX` returns the *thread-special* tran index for vacuum workers (each vacuum worker has its own `LOG_TDES` separate from the system tdes). Centralising the lookup here means every other function in the family transparently picks up the vacuum-worker tdes when called from a vacuum context — no per-callsite branching.

When a vacuum worker enters `log_sysop_start`, the function detects the vacuum context via `VACUUM_IS_THREAD_VACUUM` and emits a `vacuum_er_log` trace line (`log_manager.c:3633-3643`). Symmetric trace at end-final (`:3777-3789`).

## Concurrency — the `lock_topop` mutex

`log_sysop_start` calls `tdes->lock_topop()` on entry; `log_sysop_end_final` calls `tdes->unlock_topop()` on exit. The mutex protects the topops stack and serialises **same-tdes** sysop lifecycles — relevant only for vacuum workers and other multi-thread-per-tdes situations. Single-threaded user transactions never contend.

## Checkpoint trigger

`log_sysop_end_final` checks `LOG_ISCHECKPOINT_TIME()` and, in `SERVER_MODE`, wakes the checkpoint daemon. In `SA_MODE` it runs `logpb_checkpoint` synchronously **only** when the outermost sysop is ending (`!tdes->is_under_sysop()`) — the comment `log_manager.c:8801-8807` warns that `tdes` may be cleared by a checkpoint, so running it inside a nested sysop is unsafe.

## Failure-mode recap

| Scenario | Detection | Outcome |
|---|---|---|
| `log_sysop_commit` with empty sysop | `tail_lsa <= lastparent_lsa` | Short-circuit: pop, no record emitted |
| `log_sysop_abort` with empty sysop | same | Short-circuit: pop, no record emitted |
| `log_sysop_commit` of a logical sysop with empty body | `assert (!LSA_ISNULL (...))` at `:3853` | Hard-fail in debug; would emit a no-op record in release. Workaround: emit a dummy log record before the commit |
| Crash mid-atomic-sysop | `LOG_SYSOP_ATOMIC_START` without matching end | Recovery analysis pass schedules immediate rollback, before postpones |
| Crash after `LOG_SYSOP_START_POSTPONE` but before `LOG_SYSOP_END` | start-postpone seen, end missing | Recovery resumes the postpone replay via `log_sysop_end_recovery_postpone` |
| `log_sysop_attach_to_outer` with no parent and active TX | `topops.last == 0 && LOG_ISTRAN_ACTIVE` false | `assert_release(false)` then degrade to `log_sysop_commit` (`:4110-4115`) |
| Random crash injection (`FI_TEST_LOG_MANAGER_RANDOM_EXIT_AT_END_SYSTEMOP`) | 1/5000 probability per `log_sysop_end_random_exit` call | Process exits; recovery harness verifies invariants |

## Performance counters touched

- `PSTAT_TRAN_NUM_START_TOPOPS` — incremented in `log_sysop_start`
- `PSTAT_TRAN_NUM_END_TOPOPS` — incremented in `log_sysop_end_final`
- `PSTAT_TRAN_NUM_TOPOP_PPCACHE_HITS` / `PSTAT_TRAN_NUM_TOPOP_PPCACHE_MISS` — postpone-cache outcomes in `log_sysop_do_postpone`

## Cross-references inside CUBRID

The `log_sysop_*` family is consumed by virtually every module that mutates persistent state at greater-than-single-page granularity. Notable callers (from `grep -ln "log_sysop_" src/`):

- `src/storage/btree.c`, `btree_load.c` — page splits/merges, online index build
- `src/storage/heap_file.c`, `overflow_file.c` — multi-page heap ops, OID overflow
- `src/storage/disk_manager.c`, `extendible_hash.c` — file-extent allocation
- `src/storage/external_sort.c` — sort run produce/consume
- `src/transaction/locator_sr.c` — server-side locator (catalog updates)
- `src/transaction/log_recovery.c`, `log_2pc.c` — recovery scanner consumers
- `src/transaction/transaction_sr.c` — savepoint integration
- `src/query/serial.c` — serial-object increment ops
- `src/query/query_executor.c` — DML statement-level wrapping

## Invariants

> [!key-insight] The five hard invariants
> 1. **`log_sysop_start` always pairs with exactly one of `_commit` / `_commit_internal`-derived / `_abort` / `_attach_to_outer`** — leaking a sysop is asserted against on transaction end.
> 2. **`tdes->topops.last` advances strictly +1 / −1 per call** — random-position pop is forbidden; `log_sysop_attach_to_outer` is the only "no-end-record" path and still pops.
> 3. **The `lastparent_lsa` captured at start is the only legal rollback target for the matching abort** — modifying `tail_lsa` between start and abort by a third party (concurrent thread on the same TDES) is forbidden by `lock_topop`.
> 4. **Logical-end variants forbid empty bodies** — they assert at least one log record was appended between start and end.
> 5. **Atomic sysops are recovered eagerly** — found incomplete during analysis ⇒ rolled back unconditionally before the redo phase advances. Other sysops are recovered lazily (whatever happens to be in the WAL gets replayed in order).

## Diagnostics

The `db_serverlog/<db>/cubrid.err` log emits `vacuum_er_log` trace lines at sysop-start and sysop-end-final for vacuum-worker contexts. `er_set` calls in `log_sysop_start` and `_end_begin` produce `ER_LOG_UNKNOWN_TRANINDEX` / `ER_LOG_NOTACTIVE_TOPOPS` if the TDES lookup or topop precondition fails.

For runtime introspection, `log_check_system_op_is_started` is callable as an assertion helper from any subsystem and `log_sysop_get_level` returns the current nesting depth.

For log-stream debugging, `xlog_dump` (`log_manager.h:183`) reads each log record and `log_sysop_end_type_string` formats the end-record subtype identifier.

## Related decisions / PRs

No dedicated ADR currently. Fold candidates:

- The dual representation of "complex undo" as either a single `LOG_COMPENSATE` record vs a sysop wrapped with `LOG_SYSOP_END_LOGICAL_COMPENSATE` is a recurring design question — promotion of the latter to the former when the operation is single-page is the pattern.
- `LOG_SYSOP_ATOMIC_START` introduction predates the current source baseline; rationale lives in commit history rather than wiki.
