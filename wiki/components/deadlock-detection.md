---
type: component
parent_module: "[[components/transaction|transaction]]"
path: "src/transaction/wait_for_graph.c, wait_for_graph.h (legacy); lock_manager.c (active)"
status: active
purpose: "Deadlock detection via wait-for graph and cycle resolution"
key_files:
  - "wait_for_graph.c (WFG_NODE, WFG_EDGE, DFS cycle finder — compiled only with ENABLE_UNUSED_FUNCTION)"
  - "wait_for_graph.h (WFG public API)"
  - "lock_manager.c (active deadlock daemon, embedded cycle detection)"
public_api:
  - "lock_detect_deadlock() — embedded in lock_manager.c daemon"
  - "wfg_detect_cycle() — legacy API in wait_for_graph.c"
tags:
  - component
  - cubrid
  - deadlock
  - transaction
  - concurrency
related:
  - "[[components/transaction|transaction]]"
  - "[[components/lock-manager|lock-manager]]"
created: 2026-04-23
updated: 2026-05-08
---

# Deadlock Detection

CUBRID detects deadlocks by scanning the wait-for graph among active transactions. The **active** implementation is embedded in `lock_manager.c` and runs as a periodic daemon (`lock_deadlock_detect_daemon`). The historical `wait_for_graph.c` provides a standalone WFG library compiled only under `#if defined(ENABLE_UNUSED_FUNCTION)`.

## Wait-For Graph Concepts

A **wait-for graph** (WFG) has one node per transaction. A directed edge `T_i → T_j` means "transaction `T_i` is waiting for a lock held by transaction `T_j`." A **deadlock** is a cycle in this graph.

## `wait_for_graph.c` Data Structures

Even though this file is currently gated behind `ENABLE_UNUSED_FUNCTION`, its structures document the design:

```c
struct wfg_edge {
  int waiter_tran_index;           /* index of waiting transaction */
  int holder_tran_index;           /* index of holding transaction */
  struct wfg_edge *next_holder_edge_p;
  struct wfg_edge *next_waiter_edge_p;
};

struct wfg_node {
  WFG_STACK_STATUS status;         /* NOT_VISITED / ON_STACK / OFF_STACK / RE_ON_STACK / ON_TG_CYCLE */
  int cycle_group_no;
  int (*cycle_fun)(int tran_index, void *args);  /* resolution callback; NULL → abort */
  void *args;
  WFG_EDGE *first_holder_edge_p;
  WFG_EDGE *last_holder_edge_p;
  WFG_EDGE *first_waiter_edge_p;
  WFG_EDGE *last_waiter_edge_p;
};

struct wfg_stack {
  int       wait_tran_index;       /* current DFS node */
  WFG_EDGE *current_holder_edge_p; /* current edge being explored */
};
```

The `WFG_STACK` implements an **iterative (non-recursive) DFS** — important because the transaction count can be large and stack overflow from recursion would be catastrophic.

## DFS Cycle Finding

The algorithm uses `WFG_STACK_STATUS` to track visited state:

1. `WFG_NOT_VISITED` → start DFS from this node
2. `WFG_ON_STACK` → node is on the current DFS path → **cycle detected**
3. `WFG_OFF_STACK` → fully processed (no cycle through this node)
4. `WFG_RE_ON_STACK` / `WFG_ON_TG_CYCLE` → used during transaction-group (TG) cycle resolution

> [!key-insight] Cycle pruning
> `WFG_PRUNE_CYCLES_IN_CYCLE_GROUP = 10` caps cycles reported per group. `WFG_MAX_CYCLES_TO_REPORT = 100` caps total cycles per detection run. This prevents pathological scenarios (star-shaped deadlock with N! cycle enumeration) from consuming unbounded time.

## TWFG and Victim Storage (since PR #6930)

> [!update] Deadlock storage refactor — PR #6930 (`5e12a293c`, 2026-05-07)
> The static `.bss` arrays `TWFG_edge_block[LK_MID_TWFG_EDGE_COUNT]` and `victims[LK_MAX_VICTIM_COUNT]` are gone. Both are now lock-manager-owned heap allocations on `lk_Gl`:
>
> - `lk_Gl.TWFG_edge_storage` — `LK_WFG_EDGE *`, sized `config.mid_twfg_edge_count` entries. Allocated in `lock_initialize_deadlock_detection`. Used as the initial `lk_Gl.TWFG_edge` value at the start of each `lock_detect_local_deadlock` run.
> - `lk_Gl.victims` — `LK_DEADLOCK_VICTIM *`, sized `config.max_deadlock_victims` entries. Allocated and `memset`-zeroed in `lock_initialize_deadlock_detection`.
>
> The 2-stage WFG-edge expansion in `lock_add_WFG_edge` reads its threshold values (`min/mid/max_twfg_edge_count`) from `lk_Gl.config` instead of file-local `#define`s. Defaults: 200 / 1000 / `MAX_NTRANS²`.
>
> Cleanup at end of detection now compares pointers — `if (lk_Gl.TWFG_edge != lk_Gl.TWFG_edge_storage) free_and_init (lk_Gl.TWFG_edge);` — replacing the prior counter compare against `LK_MID_TWFG_EDGE_COUNT`.

> [!key-insight] TWFG edge-count invariant
> `lock_make_runtime_config` enforces `min_twfg_edge_count < mid_twfg_edge_count <= max_twfg_edge_count`. The 2-stage expansion in `lock_add_WFG_edge` walks `min → mid → max` and copies the previous buffer into the next; if `mid <= min` or `max < mid`, the copy or the in-place reindex walks off the end of the destination. This is unreachable under defaults but matters for any custom `LK_CONFIG`.

## Active Deadlock Daemon (in `lock_manager.c`)

The production deadlock detector runs as `lock_deadlock_detect_daemon` — a `cubthread::daemon` that wakes periodically:

1. Iterates all transactions in `LOG_TDES` array that have `lockwait != NULL` and state `LOCK_SUSPENDED`.
2. For each waiting thread, follows the `res_head → holder` chain to find which transaction holds the conflicting lock.
3. Builds the transitive wait-for relationship in memory.
4. Identifies cycles by DFS (similar logic to `wait_for_graph.c`).
5. Selects victim: **youngest transaction** (highest `tran_index`, per `LK_ISYOUNGER` macro).
6. Calls `lock_resume(victim_entry, LOCK_RESUMED_DEADLOCK_TIMEOUT)` — wakes the victim thread which returns `LK_NOTGRANTED_DUE_ABORTED`.

The victim thread's caller propagates the error upward, triggering a transaction rollback via `log_abort()`.

## Victim Selection Policy

```c
/* is younger transaction? */
#define LK_ISYOUNGER(young_tranid, old_tranid)  (young_tranid > old_tranid)
```

Youngest = highest transaction ID = started most recently. This is a **wait-die** style preference: older transactions are preferred survivors. The rationale is that older transactions have done more work and aborting them is more expensive.

## Daemon Statistics

`lock_deadlock_detect_daemon_get_stats(UINT64 *statsp)` (SERVER_MODE only) exposes daemon run counts and deadlock resolution counts to the monitoring subsystem.

## Interaction with Lock Escalation

Lock escalation (many row locks → table lock) can break deadlock cycles by consolidating resources: if `T_i` and `T_j` both escalated to class locks, the conflict becomes table-vs-table which is detected and resolved sooner.

## Related

- Parent: [[components/transaction|transaction]]
- [[components/lock-manager|lock-manager]] — contains the active daemon; provides `LK_ENTRY` and `LK_RES` data accessed during detection
- Source: [[sources/cubrid-src-transaction]]
