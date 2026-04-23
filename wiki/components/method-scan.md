---
type: component
parent_module: "[[modules/src|src]]"
path: "src/method/method_scan.cpp, src/method/method_scan.hpp"
status: active
purpose: "cubscan::method::scanner — the S_METHOD_SCAN backend; reads argument tuples from a qfile list and invokes method_invoke_group per row"
key_files:
  - "method_scan.hpp — class declaration; METHOD_SCAN_ID alias"
  - "method_scan.cpp — init/open/next_scan/close implementation"
public_api:
  - "scanner::constructor() — explicit (not a real constructor; union-safe init)"
  - "scanner::init(thread_p, sig_array, list_id) → int"
  - "scanner::open() → int — opens qfile list scan + calls method_group->begin()"
  - "scanner::next_scan(val_list_node &) → SCAN_CODE"
  - "scanner::close() → int"
  - "scanner::clear(bool is_final)"
tags:
  - component
  - cubrid
  - method
  - scan
related:
  - "[[components/method|method]]"
  - "[[components/method-invoke-group|method-invoke-group]]"
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/list-file|list-file]]"
created: 2026-04-23
updated: 2026-04-23
---

# `method-scan` — `S_METHOD_SCAN` Backend

`cubscan::method::scanner` is the concrete scan implementation behind `S_METHOD_SCAN`. It is typedef'd as `METHOD_SCAN_ID` and lives inside the `SCAN_ID.s.msid` union member of [[components/scan-manager|scan-manager]].

## Wiring into scan-manager

```c
// scan_manager.c
struct scan_id_struct {
  SCAN_TYPE type;   // S_METHOD_SCAN
  union {
    METHOD_SCAN_ID msid;   // = cubscan::method::scanner
    ...
  } s;
};
```

`scan_open_method_scan()` in `scan_manager.c` calls `scanner::init()`. `scan_next_method_scan()` calls `scanner::next_scan()`. `scan_close_scan()` calls `scanner::close()`.

## Union-Safety Note

`scanner` has no constructor. Instead, `scan_manager.c` calls `scanner::constructor()` explicitly after placement because the struct lives in a union (a CUBRID-wide convention to avoid UB with non-trivial union members).

## `next_scan()` Sequence (per row)

```
1. next_value_array(vl)        — wire up m_dbval_list linked list to val_list_node
2. get_single_tuple()           — qfile_scan_list_next() → deserialize args into m_arg_vector
3. method_invoke_group::execute(arg_wrapper)  — dispatch call
4. clone return values: get_return_value(i) → db_value_clone → m_dbval_list[i].val
5. method_invoke_group::reset(false) — clear m_result_vector for next row
6. db_value_clear on m_arg_vector    — prepare for next row
```

If `execute()` returns an error, `scan_code = S_ERROR` and `ER_SP_EXECUTE_ERROR` is set (unless the error is `ER_SM_INVALID_METHOD_ENV`, which is suppressed for CAS-side builtin methods — noted as a FIXME in source).

## Argument Source: qfile List

Method arguments arrive **pre-computed** in a `qfile_list_id` (temp list file). This list is the output of a `CSELECT` (correlated select) evaluated in the XASL tree. The scanner reads one tuple per `get_single_tuple()` call via `qfile_scan_list_next()`, then deserializes each column into `m_arg_vector[i]` using `pr_type->data_readval()`.

This means method arguments are **evaluated by the query engine before the scan call** — the scanner only reads pre-packed values. Complex expressions (subqueries, aggregates) as method arguments are handled upstream in XASL evaluation.

## Memory Layout

| Field | Allocation | Lifetime |
|---|---|---|
| `m_arg_vector` | `db_private_alloc` in `init()` | Until `clear(is_final=true)` |
| `m_arg_dom_vector` | `db_private_alloc` in `init()` | Until `clear(is_final=true)` |
| `m_dbval_list` | `db_private_alloc` in `init()` | Freed in `close_value_array()` on each `close()` |
| Each `m_dbval_list[i].val` | `db_private_alloc` per `next_scan()` | Owned by query executor until next call |
| `m_method_group` | `new` in `init()` | `delete` in `clear(is_final=true)` |

## Related

- [[components/method|method]] — parent hub
- [[components/method-invoke-group|method-invoke-group]] — `m_method_group` member
- [[components/scan-manager|scan-manager]] — contains `METHOD_SCAN_ID msid` in `SCAN_ID` union
- [[components/list-file|list-file]] — argument tuples live in a `qfile_list_id`
- [[components/query-executor|query-executor]] — drives the scan loop
