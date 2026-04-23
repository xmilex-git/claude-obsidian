---
type: component
parent_module: "[[modules/src|src]]"
path: "src/xasl/xasl_analytic.hpp"
status: active
purpose: "Analytic / window function nodes (ANALYTIC_TYPE, ANALYTIC_EVAL_TYPE) — linked groups of window functions with PARTITION BY / ORDER BY sort lists and runtime per-group tuple counters"
key_files:
  - "src/xasl/xasl_analytic.hpp (analytic_list_node, analytic_eval_type, analytic_function_info)"
tags:
  - component
  - cubrid
  - xasl
  - analytic
  - window-function
related:
  - "[[components/xasl|xasl]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/xasl-aggregate|xasl-aggregate]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/aggregate-analytic|aggregate-analytic]]"
created: 2026-04-23
updated: 2026-04-23
---

# `ANALYTIC_TYPE` — Analytic / Window Function Node

`ANALYTIC_TYPE` (typedef for `cubxasl::analytic_list_node`) represents one window function in a query. Nodes form a singly-linked list; grouped into `ANALYTIC_EVAL_TYPE` "evaluation groups" by compatible PARTITION BY / ORDER BY sort lists. The eval groups are hung off `BUILDLIST_PROC_NODE.a_eval_list`.

## `analytic_list_node`

```cpp
struct analytic_list_node {
  analytic_list_node *next;

  /* --- Serialised fields (XASL stream) --- */
  FUNC_CODE       function;          // ROW_NUMBER, RANK, DENSE_RANK, LEAD, LAG,
                                     // NTH_VALUE, NTILE, SUM, COUNT, AVG, STDDEV,
                                     // PERCENTILE_CONT / DISC, CUME_DIST, PERCENT_RANK…
  QUERY_OPTIONS   option;            // DISTINCT / ALL
  tp_domain      *domain;            // result domain
  tp_domain      *original_domain;
  DB_TYPE         opr_dbtype;
  regu_variable_node operand;        // the window function argument (inline, not pointer)
  int             flag;
  int             sort_prefix_size;  // number of PARTITION BY cols in sort list
  int             sort_list_size;    // total sort list (PARTITION BY + ORDER BY)
  int             offset_idx;        // select-list index for LEAD/LAG/NTH_VALUE offset
  int             default_idx;       // select-list index for LEAD/LAG default value
  bool            from_last;         // NTH_VALUE FROM LAST
  bool            ignore_nulls;      // RESPECT/IGNORE NULLS
  bool            is_const_operand;  // MEDIAN constant optimisation

  /* --- Runtime values (server only, not serialised) --- */
  analytic_function_info info;       // NTILE / percentile / cume_percent state
  qfile_list_id  *list_id;           // DISTINCT handling
  qfile_list_id  *group_list_id;     // group header file
  qfile_list_id  *order_list_id;     // group values file
  int             curr_group_tuple_count;
  int             curr_group_tuple_count_nn;  // non-NULL count
  int             curr_sort_key_tuple_count;
  db_value       *value;             // current window result
  db_value       *value2;            // STDDEV / VARIANCE secondary
  db_value       *out_value;         // output DB_VALUE
  db_value        part_value;        // partition accumulator
  INT64           curr_cnt;
  bool            is_first_exec_time;
};
```

## `analytic_eval_type` (evaluation group)

Functions that share the same PARTITION BY + ORDER BY keys are grouped into one `analytic_eval_type` to share a single sort pass:

```cpp
struct analytic_eval_type {
  analytic_eval_type  *next;
  analytic_list_node  *head;         // list of window functions in this group
  SORT_LIST           *sort_list;    // partition sort spec
  int                  sort_list_size;
  int                  covered_size;
  DB_VALUE            *current_values; // current partition values
  DB_VALUE            *temp_values;
};
```

## Per-function runtime info

```cpp
union analytic_function_info {
  analytic_ntile_function_info ntile;
    // { bool is_null; int bucket_count; }

  analytic_percentile_function_info percentile;
    // { double cur_group_percentile; regu_variable_node *percentile_reguvar; }

  analytic_cume_percent_function_info cume_percent;
    // { int last_pos; double last_res; }
};
```

## Serialised vs runtime split

> [!key-insight] Same pattern as aggregate nodes
> Fields up to `is_const_operand` are serialised. Everything from `info` downward is server-only runtime state. `qfile_list_id` pointers are allocated on the server at execution time. This is identical to the split in `AGGREGATE_TYPE` — a consistent pattern across all XASL node types.

## `sort_prefix_size` vs `sort_list_size`

- `sort_prefix_size` = number of PARTITION BY columns at the front of the sort list.
- `sort_list_size` = total columns (PARTITION BY + ORDER BY).
- Columns in `[sort_prefix_size, sort_list_size)` are the ORDER BY keys within each partition.

## Namespace and aliases

```cpp
using ANALYTIC_TYPE                     = cubxasl::analytic_list_node;
using ANALYTIC_EVAL_TYPE                = cubxasl::analytic_eval_type;
using ANALYTIC_FUNCTION_INFO            = cubxasl::analytic_function_info;
using ANALYTIC_NTILE_FUNCTION_INFO      = cubxasl::analytic_ntile_function_info;
using ANALYTIC_PERCENTILE_FUNCTION_INFO = cubxasl::analytic_percentile_function_info;
using ANALYTIC_CUME_PERCENT_FUNCTION_INFO = cubxasl::analytic_cume_percent_function_info;
```

## Related

- [[components/xasl|xasl]] — `BUILDLIST_PROC_NODE.a_eval_list`; `a_regu_list`, `a_scan_regu_list`, `a_outptr_list`
- [[components/regu-variable|regu-variable]] — `operand` is an inline `regu_variable_node`
- [[components/xasl-aggregate|xasl-aggregate]] — aggregate functions (separate node type, different accumulator)
- [[components/aggregate-analytic|aggregate-analytic]] — server-side evaluation driver
- [[components/query-executor|query-executor]] — drives the sort + frame evaluation loop
- Source: [[sources/cubrid-src-xasl|cubrid-src-xasl]]
