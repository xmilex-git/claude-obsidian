---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/set_scan.c + set_scan.h"
status: active
purpose: "Iterate the elements of a DB_SET, DB_MULTISET, or DB_SEQUENCE value as if it were a one-column virtual table (S_SET_SCAN in scan_manager)"
key_files:
  - "src/query/set_scan.c — single function qproc_next_set_scan()"
  - "src/query/set_scan.h — public prototype"
  - "src/query/scan_manager.h — SET_SCAN_ID embedded in SCAN_ID.s.ssid"
public_api:
  - "qproc_next_set_scan(thread_p, s_id) → SCAN_CODE"
tags:
  - component
  - cubrid
  - query
  - scan
  - set
related:
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/db-value|db-value]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `set_scan.c` — Set / Multiset / Sequence Scan

`set_scan.c` is the smallest scan implementation in CUBRID: a single 158-line function `qproc_next_set_scan` that turns a `DB_VALUE` holding a `DB_SET`, `DB_MULTISET`, or `DB_SEQUENCE` into a sequential scan source. It covers the `S_SET_SCAN` scan type in [[components/scan-manager|scan-manager]]'s 15-type union.

## Purpose

CUBRID's SQL supports set-typed columns (`SET OF`, `MULTISET OF`, `SEQUENCE OF`). When a query unnests a set (e.g. `SELECT * FROM TABLE(set_column) AS t`) or iterates the elements of an inline set literal, the executor opens a `SET_SCAN_ID` and calls `qproc_next_set_scan` on each iteration.

## Public Entry Points

| Function | Phase | Description |
|---|---|---|
| `qproc_next_set_scan(thread_p, s_id)` | next | Returns `S_SUCCESS`, `S_END`, or `S_ERROR`; advances `s_id->s.ssid.cur_index` by 1 on each call |

There is no dedicated `open`, `start`, or `close` function for set scans. Initialization of `SET_SCAN_ID` is handled by the generic `scan_open_set_scan()` path in `scan_manager.c`, and cleanup is trivial (no allocated resources beyond the `SCAN_ID` itself).

## Execution Path

```
scan_manager (S_SET_SCAN)
  └── qproc_next_set_scan(thread_p, s_id)
        ├── [S_BEFORE → S_ON]  set cur_index = 0; fetch element 0
        ├── [S_ON]             increment cur_index; check against set_card
        └── [S_AFTER]          return S_END
```

The function dispatches on `s_id->s.ssid.set_ptr->type`:

| REGU_VARIABLE type | Code path |
|---|---|
| `TYPE_FUNC` + `F_SEQUENCE` | Walks `ssid.operand` linked-list of `REGU_VARIABLE_LIST` nodes; calls `fetch_copy_dbval` on each node's `value` into `s_id->val_list->valp->val` |
| Everything else (SET / MULTISET) | Calls `db_get_set()` on `ssid.set` to get `DB_SET*`; calls `db_set_get(setp, cur_index, ...)` for indexed access |

## S_SET_SCAN Semantics

### SET_SCAN_ID layout (inside SCAN_ID)
```c
// from scan_manager.h:
typedef struct set_scan_id SET_SCAN_ID;
struct set_scan_id {
  REGU_VARIABLE *set_ptr;   // points to the set expression (TYPE_FUNC/F_SEQUENCE or a set-typed attr)
  REGU_VARIABLE_LIST operand; // for F_SEQUENCE: head of the operand list
  DB_VALUE set;             // for SET/MULTISET: the materialized DB_VALUE
  int set_card;             // cardinality (db_set_size)
  int cur_index;            // current element index (0-based)
};
```

### Two distinct iteration modes

**F_SEQUENCE mode** (inline sequence literal):
- `operand` is a `REGU_VARIABLE_LIST` linked list. Each element is fetched by `fetch_copy_dbval` into the output `val_list->valp->val`. The list is advanced via `operand = operand->next`.
- `set_card` is pre-computed but the termination is actually detected by `operand == NULL` when `position == S_BEFORE`.

**DB_SET / DB_MULTISET mode**:
- `ssid.set` is a `DB_VALUE` of type `DB_TYPE_SET` or `DB_TYPE_MULTISET`.
- `db_get_set()` extracts the `DB_SET*` handle; `db_set_get(setp, cur_index, out)` retrieves element by index.
- `db_set_size(setp)` gives `set_card` (re-computed each call since it is retrieved from the container).

> [!key-insight] Order guarantees differ by type
> `SEQUENCE` (F_SEQUENCE) preserves insertion order exactly — it walks the linked list. `SET` and `MULTISET` use `db_set_get(index)` which is position-stable within a single scan but the physical element ordering inside `DB_SET` is an implementation detail of the object layer (`db_set.c`). The SQL standard does not guarantee a specific order for SET/MULTISET elements; callers should not rely on it.

## Interaction with DB_VALUE SET Encoding

- `DB_TYPE_SET` and `DB_TYPE_MULTISET` are encoded as `DB_VALUE.data.set` (`DB_SET*`), a heap-allocated container managed by `src/compat/db_set.c`.
- `DB_TYPE_SEQUENCE` literals in SQL map to `REGU_VARIABLE.type = TYPE_FUNC, ftype = F_SEQUENCE` with operands as a linked list — they never materialise a `DB_SET` at scan time.
- `fetch_copy_dbval` is used for F_SEQUENCE to evaluate each operand in the current value descriptor context, supporting computed expressions inside a sequence literal.

> [!warning] Empty-set handling is asymmetric
> For F_SEQUENCE, an empty sequence is detected by `ssid.operand == NULL` at the `S_BEFORE` check — it returns `S_END` immediately without entering `S_ON`. For DB_SET/MULTISET, emptiness is detected by `db_set_size(setp) == 0`. A NULL `DB_VALUE` in `ssid.set` is also caught (`DB_IS_NULL(&set_id->set)`) and returns `S_END`. Mismatch between these two checks is a potential gotcha when reading the code.

## Constraints

- **No parallel support**: `qproc_next_set_scan` is purely single-threaded; set elements are scanned one by one with no parallelism.
- **No spill / no memory budget**: All elements are in-memory in the `DB_SET` object or in the REGU_VARIABLE list.
- **scan_op_type**: The set scan only supports forward sequential access (`S_BEFORE → S_ON → S_AFTER`). Positional jumps are not implemented; calling with `s_id->position == S_AFTER` returns `S_END`.
- **No filter predicates at scan level**: Filtering is applied by the caller (scan_manager / query_executor) after `qproc_next_set_scan` returns `S_SUCCESS`.
- **Server + SA_MODE only**: `scan_manager.h` and `SCAN_ID` mechanics are server-side only.

## Lifecycle

```
open  : scan_open_set_scan() in scan_manager.c
          — populates SET_SCAN_ID.set_ptr, .operand, .set, .set_card, .cur_index=0
          — sets s_id->position = S_BEFORE
start : (none — set scan has no start function; scan_start_scan is a no-op for S_SET_SCAN)
next  : qproc_next_set_scan() — called in a loop by scan_manager
          — first call: S_BEFORE → S_ON, fetch element 0
          — each subsequent call: advance cur_index, fetch element
          — returns S_END when cur_index == set_card
close : scan_close_scan() in scan_manager.c — frees SCAN_ID resources;
          set_id->set DB_VALUE is cleared via pr_clear_value
```

Per-element cost: one `db_set_get` (O(1) indexed lookup) or one `fetch_copy_dbval` (F_SEQUENCE), plus one output `DB_VALUE` copy.

## Related

- [[components/scan-manager|scan-manager]] — dispatches `S_SET_SCAN` to `qproc_next_set_scan`; holds `SET_SCAN_ID` inside `SCAN_ID.s.ssid`
- [[components/query-executor|query-executor]] — generates `S_SET_SCAN` specs for set-typed derived tables
- [[components/db-value|db-value]] — `DB_VALUE`, `DB_TYPE_SET`, `DB_TYPE_MULTISET`, `DB_TYPE_SEQUENCE` encoding
- [[components/regu-variable|regu-variable]] — `TYPE_FUNC` / `F_SEQUENCE` regu variable type that drives the sequence path
