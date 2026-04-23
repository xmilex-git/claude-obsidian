---
type: component
parent_module: "[[modules/src|src]]"
path: "src/xasl/xasl_aggregate.hpp"
status: active
purpose: "Aggregate function node (AGGREGATE_TYPE / aggregate_list_node) — linked list of aggregate functions attached to BUILDLIST_PROC or BUILDVALUE_PROC XASL nodes; holds runtime accumulator split from client-serialised fields"
key_files:
  - "src/xasl/xasl_aggregate.hpp (aggregate_list_node, aggregate_accumulator, aggregate_percentile_info)"
tags:
  - component
  - cubrid
  - xasl
  - aggregate
related:
  - "[[components/xasl|xasl]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/xasl-analytic|xasl-analytic]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/aggregate-analytic|aggregate-analytic]]"
created: 2026-04-23
updated: 2026-04-23
---

# `AGGREGATE_TYPE` — Aggregate Function Node

`AGGREGATE_TYPE` (typedef for `cubxasl::aggregate_list_node`) represents one aggregate function in a `GROUP BY` or aggregate-only query. Nodes are chained into a singly-linked list hung off `BUILDLIST_PROC_NODE.g_agg_list` or `BUILDVALUE_PROC_NODE.agg_list`.

## Structure

```cpp
struct aggregate_list_node {
  aggregate_list_node *next;           // linked list
  tp_domain           *domain;         // result domain
  tp_domain           *original_domain;// pre-clone domain
  FUNC_CODE            function;       // COUNT, SUM, AVG, MIN, MAX, GROUP_CONCAT,
                                       // PERCENTILE_CONT, PERCENTILE_DISC, CUME_DIST, …
  QUERY_OPTIONS        option;         // DISTINCT / ALL
  DB_TYPE              opr_dbtype;     // operand type
  regu_variable_list_node *operands;   // function arguments (REGU_VARIABLE list)
  qfile_list_id       *list_id;        // for DISTINCT handling
  BTID                 btid;           // B-tree for min/max optimisation
  SORT_LIST           *sort_list;      // ORDER BY inside GROUP_CONCAT
  aggregate_specific_function_info info; // PERCENTILE / CUME_DIST / PERCENT_RANK data
  aggregate_accumulator accumulator;   // runtime: value, value2, curr_cnt
  // SERVER_MODE | SA_MODE only:
  aggregate_accumulator_domain accumulator_domain; // runtime domain info
  struct {
    bool agg_optimized;        // aggregate is optimized
    bool min_max_optimized;    // min/max via index
    bool part_key_descending;  // partitioning key is descending
    bool dummy;
  } flag;
  int is_ended;
};
```

## Runtime accumulator vs serialised fields

> [!key-insight] Split between client and server
> Fields above `accumulator` are serialised (packed into the XASL stream). The `aggregate_accumulator` and `aggregate_accumulator_domain` are **server-only runtime state** — they are initialised at execution time, not sent over the wire. `accumulator.value` and `accumulator.value2` hold the running sum / count / std-dev working values.

```cpp
struct aggregate_accumulator {
  db_value *value;   // primary running value
  db_value *value2;  // secondary (GROUP_CONCAT, STDDEV, VARIANCE)
  INT64     curr_cnt;// current row count
  bool clear_value_at_clone_decache;
  bool clear_value2_at_clone_decache;
};
```

## Special-function info

```cpp
union aggregate_specific_function_info {
  aggregate_percentile_info percentile;    // PERCENTILE_CONT / PERCENTILE_DISC
  // SERVER_MODE | SA_MODE:
  aggregate_dist_percent_info dist_percent; // CUME_DIST / PERCENT_RANK
};

struct aggregate_percentile_info {
  double cur_group_percentile;
  regu_variable_node *percentile_reguvar;  // the percentile value expr
};
```

`CUME_DIST` / `PERCENT_RANK` runtime arrays (`const_array`, `list_len`, `nlargers`) are server-only and not serialised.

## Optimisation flags

- `min_max_optimized` — set when MIN/MAX can be satisfied by reading the first/last entry of a B-tree (`btid` field used). The `btid` is serialised.
- `agg_optimized` — set when the aggregate is otherwise pushed into the access path.
- Hash aggregate evaluation: `BUILDLIST_PROC_NODE.g_hash_eligible` + `BUILDLIST_PROC_NODE.agg_hash_context` (not in this header — see [[components/aggregate-analytic|aggregate-analytic]]).

## Namespace and aliases

```cpp
// in namespace cubxasl
using AGGREGATE_TYPE                 = cubxasl::aggregate_list_node;
using AGGREGATE_ACCUMULATOR          = cubxasl::aggregate_accumulator;
using AGGREGATE_PERCENTILE_INFO      = cubxasl::aggregate_percentile_info;
using AGGREGATE_SPECIFIC_FUNCTION_INFO = cubxasl::aggregate_specific_function_info;
// SERVER|SA only:
using AGGREGATE_DIST_PERCENT_INFO    = cubxasl::aggregate_dist_percent_info;
using AGGREGATE_ACCUMULATOR_DOMAIN   = cubxasl::aggregate_accumulator_domain;
```

## Related

- [[components/xasl|xasl]] — `BUILDLIST_PROC_NODE.g_agg_list` / `BUILDVALUE_PROC_NODE.agg_list`
- [[components/regu-variable|regu-variable]] — `operands` is a `REGU_VARIABLE_LIST`
- [[components/xasl-analytic|xasl-analytic]] — window/analytic functions (different node type)
- [[components/aggregate-analytic|aggregate-analytic]] — server-side GROUP BY and window evaluation
- [[components/query-executor|query-executor]] — drives accumulation per group
- Source: [[sources/cubrid-src-xasl|cubrid-src-xasl]]
