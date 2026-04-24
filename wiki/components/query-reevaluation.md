---
created: 2026-04-23
type: component
title: "query-reevaluation — MVCC Predicate Re-Evaluation"
parent_module: "[[modules/src|src]]"
path: src/query/query_reevaluation.{cpp,hpp}
status: developing
key_files:
  - src/query/query_reevaluation.cpp
  - src/query/query_reevaluation.hpp
public_api:
  - cubquery::mvcc_reev_data
  - cubquery::upddel_mvcc_cond_reeval
  - cubquery::mvcc_update_reev_data
  - cubquery::mvcc_scan_reev_data
tags:
  - cubrid
  - mvcc
  - reevaluation
  - concurrency
  - server-side
  - cpp
---

# query-reevaluation — MVCC Predicate Re-Evaluation

> [!key-insight]
> REEVAL is triggered when a row passed the **scan-time predicate** on an **old MVCC version** but by the time the executor tries to lock/update it, the row has been modified by a concurrent transaction. The executor must re-fetch the **current version** of the row and re-run all predicates. If the re-evaluated predicate fails, the row is **silently skipped** (for SELECT) or the operation is aborted (for UPDATE/DELETE depending on isolation level).

## Purpose

`query_reevaluation.cpp` (tiny file, ~165 lines) defines the C++ data structures that carry the re-evaluation context through the scan manager and executor. It lives in the `cubquery` namespace and provides:

1. **`mvcc_reev_data`** — top-level union discriminated by `MVCC_REEV_DATA_TYPE`:
   - `REEV_DATA_UPDDEL`: for UPDATE / DELETE; carries `mvcc_update_reev_data*`
   - `REEV_DATA_SCAN`: for SELECT; carries `mvcc_scan_reev_data*`
2. **`upddel_mvcc_cond_reeval`** — per-class reevaluation data: filter copies (range/key/data), class OID, instance OID, rest-attrs for non-predicate columns.
3. **`mvcc_update_reev_data`** — UPDATE/DELETE: list of `upddel_mvcc_cond_reeval` nodes + assignment list + constraint predicate + copy area.
4. **`mvcc_scan_reev_data`** — SELECT: pointers to range/key/data filters.

---

## Key Data Structures

```cpp
namespace cubquery {

// Top-level discriminated union
struct mvcc_reev_data {
    MVCC_REEV_DATA_TYPE type;    // REEV_DATA_UPDDEL or REEV_DATA_SCAN
    union {
        mvcc_update_reev_data *upddel_reev_data;
        mvcc_scan_reev_data   *select_reev_data;
    };
    DB_LOGICAL filter_result;    // V_TRUE / V_FALSE / V_UNKNOWN after reeval

    void set_update_reevaluation(mvcc_update_reev_data&);
    void set_scan_reevaluation(mvcc_scan_reev_data&);
};

// Per-class condition reevaluation state (UPDATE/DELETE)
struct upddel_mvcc_cond_reeval {
    int class_index;             // index in select list
    OID cls_oid;
    OID *inst_oid;               // OID of instance being re-evaluated
    filter_info data_filter;     // copy of data predicate + attrs
    filter_info key_filter;      // copy of key filter (index scan only)
    filter_info range_filter;    // copy of range filter (index scan only)
    QPROC_QUALIFICATION qualification; // in+out: QUALIFIED / NOT_QUALIFIED
    regu_variable_list_node *rest_regu_list;
    scan_attrs *rest_attrs;
    upddel_mvcc_cond_reeval *next;

    void init(scan_id_struct& sid);  // populate from S_HEAP_SCAN or S_INDX_SCAN
};

// UPDATE/DELETE full reeval context
struct mvcc_update_reev_data {
    upddel_mvcc_cond_reeval *mvcc_cond_reev_list;   // all classes in condition
    upddel_mvcc_cond_reeval *curr_upddel;            // class being updated/deleted
    int curr_extra_assign_cnt;
    upddel_mvcc_cond_reeval **curr_extra_assign_reev; // classes in right-side assigns
    update_mvcc_reev_assignment *curr_assigns;        // assignment list
    heap_cache_attrinfo *curr_attrinfo;
    cubxasl::pred_expr *cons_pred;                   // constraint predicate
    lc_copy_area *copyarea;                          // new record buffer
    val_descr *vd;
    recdes *new_recdes;                              // built new record
};

// SELECT reeval context (just filter pointers)
struct mvcc_scan_reev_data {
    filter_info *range_filter;   // NULL if not index scan
    filter_info *key_filter;     // NULL if not index scan
    filter_info *data_filter;
    QPROC_QUALIFICATION *qualification;

    void set_filters(upddel_mvcc_cond_reeval& ureev);
};

} // namespace cubquery
```

---

## When REEVAL Is Triggered

```
Scan iteration (heap or index scan):
    lock_object_on_iscan / heap_get_visible_version
        → row locked or fetched successfully: REEVAL not needed
        → ER_MVCC_ROW_UPDATED_BY_ME or concurrent update detected:
              set reeval context
              re-fetch current version (heap_get_last_version)
              re-run all predicates via mvcc_scan_reev_data or mvcc_update_reev_data
              if filter_result == V_FALSE: skip row (SELECT) or skip/abort (UPDATE/DELETE)
              if filter_result == V_TRUE:  proceed with operation on new version
```

> [!key-insight]
> REEVAL is a READ-COMMITTED isolation semantic. Under REPEATABLE READ or SERIALIZABLE, the query engine typically aborts the entire transaction instead of silently skipping the updated row. The `QPROC_QUALIFICATION` output field distinguishes between `QPROC_NOT_QUALIFIED` (skip), `QPROC_QUALIFIED` (proceed), and `QPROC_INVALID_TUPLOID` (row deleted — always skip).

---

## `upddel_mvcc_cond_reeval::init` — Scan-Type Dispatch

```cpp
void upddel_mvcc_cond_reeval::init(scan_id_struct& sid) {
    switch (sid.type) {
    case S_HEAP_SCAN:
        range_filter = filter_info();   // empty (heap has no range filter)
        key_filter   = filter_info();   // empty
        data_filter  = { &sid.s.hsid.scan_pred, &sid.s.hsid.pred_attrs, … };
        rest_attrs   = &sid.s.hsid.rest_attrs;
        break;
    case S_INDX_SCAN:
        range_filter = { &sid.s.isid.range_pred, &sid.s.isid.range_attrs, … };
        key_filter   = { &sid.s.isid.key_pred,   &sid.s.isid.key_attrs,  … };
        data_filter  = { &sid.s.isid.scan_pred,  &sid.s.isid.pred_attrs, … };
        rest_attrs   = &sid.s.isid.rest_attrs;
        break;
    case S_PARALLEL_HEAP_SCAN:
        assert(0);    // parallel scans do not support REEVAL
        break;
    }
}
```

> [!warning]
> `S_PARALLEL_HEAP_SCAN` hits `assert(0)` in `upddel_mvcc_cond_reeval::init`. Parallel heap scans and MVCC reevaluation are **mutually exclusive** in the current implementation. Any UPDATE/DELETE with a parallel scan configuration will fail in debug builds.

---

## `mvcc_scan_reev_data::set_filters`

Copies filter pointers from an `upddel_mvcc_cond_reeval` into the scan reeval data, eliding NULL filters (predicates with no regu_list):

```cpp
void mvcc_scan_reev_data::set_filters(upddel_mvcc_cond_reeval& ureev) {
    range_filter = (ureev.range_filter.scan_pred->regu_list != NULL) ? &ureev.range_filter : NULL;
    key_filter   = (ureev.key_filter.scan_pred->regu_list   != NULL) ? &ureev.key_filter   : NULL;
    data_filter  = (ureev.data_filter.scan_pred->regu_list  != NULL) ? &ureev.data_filter  : NULL;
}
```

This ensures that filters without active predicates are not re-evaluated (short-circuit optimization).

---

## Assignment Reevaluation (UPDATE)

For UPDATE, `mvcc_update_reev_data` also carries `curr_assigns` (linked list of `update_mvcc_reev_assignment`):

```cpp
struct update_mvcc_reev_assignment {
    int att_id;                          // attribute index
    db_value *constant;                  // constant value (or NULL)
    regu_variable_node *regu_right;      // expression for right side
    update_mvcc_reev_assignment *next;
};
```

After predicate reevaluation passes, assignments are re-evaluated against the **new version's attribute values** (not the version that originally matched the scan), then written into `new_recdes` via `lc_copy_area`. This ensures the update is applied to the current committed state.

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Isolation level | Primarily for READ COMMITTED; REPEATABLE READ behavior controlled by caller |
| Parallel scan | `S_PARALLEL_HEAP_SCAN` not supported (assert) |
| Memory | Structures allocated on stack by callers inside scan_manager; `lc_copy_area` heap-allocated per re-eval |
| Build mode | Server-side (SERVER_MODE + SA_MODE) |
| Namespace | `cubquery::` C++ namespace; C aliases via `using` declarations for compatibility |

---

## Lifecycle

```
Per scan cycle where concurrent update is detected:
    upddel_mvcc_cond_reeval reev_data;
    reev_data.init(scan_id);           // copy filter references
    mvcc_reev_data reev;
    reev.set_update_reevaluation(upddel_data);  // or set_scan_reevaluation

    // passed into heap/index scan:
    heap_get_last_version(oid, reev)
        → re-fetch current version
        → re-evaluate predicates
        → reev.filter_result = V_TRUE / V_FALSE / V_UNKNOWN

    // caller checks reev.filter_result:
    V_TRUE  → proceed with update/delete on current version
    V_FALSE → skip (row no longer qualifies)
    V_UNKNOWN / error → depends on isolation level; may abort transaction
```

---

## Related

- [[components/mvcc]] — MVCC snapshot, `mvcc_satisfies_snapshot`; visibility decisions that trigger reevaluation
- [[components/scan-manager]] — passes `MVCC_REEV_DATA` into heap and index scan functions
- [[components/lock-manager]] — lock acquisition failure on concurrent update triggers reevaluation path
- [[components/query-executor]] — `qexec_execute_mainblock` sets up reeval data before scan
- [[components/heap-file]] — `heap_get_last_version` re-fetches current version for reevaluation
- [[components/regu-variable]] — `regu_variable_node` in assignment list re-evaluated against new version
- [[Build Modes (SERVER SA CS)]]
