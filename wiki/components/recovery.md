---
type: component
parent_module: "[[components/transaction|transaction]]"
path: "src/transaction/log_recovery.c"
status: active
purpose: "ARIES crash recovery: analysis, redo, undo phases; CLR handling; 2PC recovery"
key_files:
  - "log_recovery.c (~15K lines: full ARIES implementation)"
  - "log_manager.h (log_initialize triggers recovery)"
  - "log_record.hpp (LOG_COMPENSATE, LOG_SYSOP_ATOMIC_START record types)"
public_api:
  - "log_rv_analysis() — analysis phase: rebuild transaction table from checkpoint"
  - "log_rv_redo_all_in_log_pages() — redo phase: replay all records from redo-start LSA"
  - "log_rv_undo_rec() — undo phase: roll back loser transactions"
  - "(all called internally from log_initialize via log_recovery_redo / log_recovery_undo)"
tags:
  - component
  - cubrid
  - recovery
  - aries
  - transaction
  - wal
related:
  - "[[components/transaction|transaction]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/server-boot|server-boot]]"
  - "[[components/storage|storage]]"
  - "[[components/page-buffer|page-buffer]]"
  - "[[components/vacuum|vacuum]]"
created: 2026-04-23
updated: 2026-04-23
---

# Crash Recovery

`src/transaction/log_recovery.c` implements ARIES (Algorithm for Recovery and Isolation Exploiting Semantics) crash recovery. It runs inside `log_initialize()` whenever the server detects that the database was not cleanly shut down.

## When Recovery Triggers

```
boot_restart_server()
  └── log_initialize(... ismedia_crash=0 ...)
        ├── Read log header
        ├── Check last checkpoint LSA vs EOF LSA
        │     If checkpoint != EOF → crash detected
        └── log_recovery(thread_p, ismedia_crash, r_args)
              ├── Phase 1: Analysis
              ├── Phase 2: Redo
              └── Phase 3: Undo
```

A `LOG_DUMMY_CRASH_RECOVERY` sentinel record (type 37) is written at the start of recovery to mark the recovery boundary in the log — useful for diagnostics and for nested recovery scenarios.

## Phase 1: Analysis

**Goal**: reconstruct the transaction table and find the correct redo-start LSA.

```
log_rv_analysis()
  Start from: last checkpoint LSA (log_header.chkpt_lsa)
  Scan forward through all log records to EOF

  For each record:
    - Rebuild LOG_TDES for active transactions
    - Track each transaction's last LSA (tail_lsa)
    - Note "losers" = transactions with no COMMIT/ABORT record
    - Determine redo_start_lsa = min(chkpt_lsa, min(tdes->begin_lsa) for all active)
```

The analysis phase processes `LOG_END_CHKPT` records to restore the initial transaction table snapshot, then replays forward from there.

## Phase 2: Redo

**Goal**: restore all committed changes (and in-flight changes) to their post-crash state.

```
log_rv_redo_all_in_log_pages()
  Start from: redo_start_lsa (computed in analysis)
  Scan forward to EOF

  For each record that has redo data:
    1. Read target page into buffer (pgbuf_fix with OLD_PAGE_IF_EXISTS)
    2. Compare page LSA vs record LSA
       If page_lsa >= record_lsa → page already has this change (skip)
       If page_lsa < record_lsa  → apply redo action
    3. Call RV_FUN[rcvindex].redo_fun(thread_p, &rcv)
    4. pgbuf_set_lsa(thread_p, page, record_lsa)
    5. pgbuf_set_dirty() + pgbuf_unfix()
```

> [!key-insight] Redo is idempotent by page LSA check
> Each redo record carries the target `(volid, pageid, offset)` and the `LOG_LSA`. The page's embedded LSA is compared before applying — if the page is already at or past the record's LSA, redo is skipped. This makes redo safe to repeat after a secondary crash during recovery.

### Atomic System Operations

`LOG_SYSOP_ATOMIC_START` (type 50) marks operations that **must be rolled back immediately** if recovery encounters them incomplete. After the redo phase, before running postpones, recovery scans for incomplete atomic sysops and undoes them. This prevents partially-executed index splits (B-tree page splits spanning multiple log records) from persisting. The marker is emitted by [[components/log-sysop|log_sysop_start_atomic]] (`log_manager.c:3665`); see that page for the full sysop family lifecycle and the six `LOG_SYSOP_END_TYPE` end-record variants.

## Phase 3: Undo

**Goal**: roll back all loser transactions (those without a `LOG_COMMIT` record).

```
log_rv_undo_all()
  For each loser transaction (in reverse begin-LSA order):
    Walk tdes->tail_lsa backward via prev_tranlsa chain
    For each undo record:
      1. Call RV_FUN[rcvindex].undo_fun(thread_p, &rcv)
      2. Write LOG_COMPENSATE (CLR) record
         CLR's undo_nxlsa = record's prev_tranlsa
         (skip over the just-undone record during any future undo scan)
    Write LOG_ABORT for the transaction
```

### Compensation Log Records (CLR)

`LOG_COMPENSATE` records prevent double-undo. The CLR's `undo_nxlsa` field points to the log record that should be processed next during undo, effectively jumping over the record that was just compensated. If recovery crashes during undo and restarts, the CLRs ensure each undo action is applied exactly once.

> [!warning] Undo order matters
> Losers are processed in reverse order of their begin LSA (newest loser first). Within a single transaction, undo records are processed strictly in reverse LSN order (following `prev_tranlsa`). Violating this order corrupts the page state.

## 2-Phase Commit (2PC) Recovery

Transactions in `LOG_2PC_PREPARE` state are "in-doubt" — recovery cannot unilaterally abort them because the distributed coordinator may have committed. Recovery leaves them in prepared state and exposes them for external resolution:

- `LOG_2PC_COMMIT_DECISION` → redo the commit
- `LOG_2PC_ABORT_DECISION` → redo the abort
- No decision record → leave in-doubt; expose via `lock_reacquire_crash_locks()`

## Media Recovery

When `ismedia_crash=1` (passed from `boot_restart_server` during `cubrid restoredb`):
- Log is replayed from a backup LSA, not a checkpoint LSA.
- Archive volumes are read sequentially via `log_page_buffer.c`.
- `log_rv_analysis` scans from the backup point forward.

## Recovery and Vacuum

After redo completes but before undo, vacuum data is also recovered:
- Vacuum's data file (`vacuum_data_vfid`) is read and validated.
- Dropped-files list is reconstructed from `dropped_files_vfid`.
- The vacuum master daemon is then started to resume GC.

> [!key-insight] Vacuum runs after redo, before normal operations
> The vacuum daemon must not start until redo is complete (all committed changes visible) but can start before undo finishes. Dead rows from loser transactions are handled correctly because `mvcc_satisfies_vacuum` checks against the `oldest_mvccid` of active snapshots, which excludes in-progress recovery transactions.

## Recovery Function Table

Each `LOG_RCVINDEX` (recovery index) maps to a pair of C functions:

```c
typedef struct rv_fun_entry RV_FUN_ENTRY;
struct rv_fun_entry {
  LOG_RCV_REDO_FUNCTION redo_fun;   /* NULL if redo-only not needed */
  LOG_RCV_UNDO_FUNCTION undo_fun;   /* NULL if undo-only not needed */
};
extern RV_FUN_ENTRY rv_fun_tab[];
```

This table (defined in `recovery.c`) is the dispatch mechanism for all redo/undo handlers. Examples:
- `RVHF_MVCC_INSERT` → heap insert redo/undo
- `RVBT_MVCC_DELETE_OBJECT` → B-tree MVCC key delete redo/undo
- `RVPGBUF_DEALLOC` → page deallocation undo

## From the Manual (admin/troubleshoot.rst, release_note 11.4 — added 2026-04-27)

> [!gap] Documented operator behaviors
> - **Log-recovery emits paired NOTIFICATION codes**: `-1128` (start, with `"log records to be applied: N, log page: A~B"`) and `-1129` (finish). Logged to `$CUBRID/log/server/<db>_<yyyymmdd>_<hhmi>.err`. The redo count + page range is how operators measure recovery time. (`admin/troubleshoot.rst:122-133`).
> - **`recovery_progress_logging_interval`** controls progress reporting cadence between `-1128`/`-1129`. (`admin/config.rst:2325-2326`).
> - **NEW 11.4: Parallel REDO recovery** — page-by-page parallel apply where no synchronization required. Most noticeable when REDO dominates recovery time and parallel index is high.
> - **HA `force_remove_log_archives` MUST be `no`** — else archive logs needed by `applylogdb` may be deleted, causing replication inconsistency.

See [[sources/cubrid-manual-admin]] · [[sources/cubrid-manual-ha]] for the full operator manual context.

## Related

- Parent: [[components/transaction|transaction]]
- [[components/log-manager|log-manager]] — provides the log records consumed during recovery
- [[components/server-boot|server-boot]] — triggers `log_initialize` which invokes recovery
- [[components/storage|storage]] — DWB (double-write buffer) prevents torn pages from being recovered incorrectly
- [[components/page-buffer|page-buffer]] — redo writes go through `pgbuf_fix` + `pgbuf_set_dirty`
- [[components/vacuum|vacuum]] — vacuum data recovered after redo phase
- Source: [[sources/cubrid-src-transaction]]
- Manual: [[sources/cubrid-manual-admin]] · [[sources/cubrid-manual-ha]]
