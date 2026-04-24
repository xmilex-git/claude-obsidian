---
created: 2026-04-23
type: component
title: "query-dump — XASL Plan Pretty-Printer (EXPLAIN)"
parent_module: "[[modules/src|src]]"
path: src/query/query_dump.{c,h}
status: developing
key_files:
  - src/query/query_dump.c
  - src/query/query_dump.h
public_api:
  - qdump_print_xasl
  - qdump_print_stats_json
  - qdump_print_stats_text
  - qdump_operator_type_string
  - qdump_default_expression_string
tags:
  - cubrid
  - query
  - explain
  - xasl
  - debug
  - json
---

# query-dump — XASL Plan Pretty-Printer (EXPLAIN)

> [!key-insight]
> **EXPLAIN output in CUBRID is produced server-side.** `qdump_print_stats_json` and `qdump_print_stats_text` are compiled only under `#if defined(SERVER_MODE)`, and they run on the server after execution. The JSON stats object is sent back to the client as part of the query trace response — the client displays it, but never generates it. `qdump_print_xasl` (stdout dump) is available in all build modes and is a debug/development tool.

## Purpose

`query_dump.c` (~4.5 K lines, ~80 static functions) decodes and renders every field of an `XASL_NODE` tree:

- **`qdump_print_xasl`**: full text dump of the XASL plan tree to `stdout` (always-available; used by CSQL `trace plan` in non-JSON mode).
- **`qdump_print_stats_json`**: SERVER_MODE; iterates the executed XASL tree and fills a `json_t*` (jansson) with per-node execution statistics (tuple counts, page I/O, etc.).
- **`qdump_print_stats_text`**: SERVER_MODE; text version of the same stats with indentation for nested subqueries.
- Helper string converters: `qdump_operator_type_string`, `qdump_default_expression_string`, etc. — used by both dump and error messages.

---

## Public Entry Points

| Signature | Role |
|-----------|------|
| `bool qdump_print_xasl(xasl_node* xasl)` | Full plan dump to stdout; debug / EXPLAIN plan text |
| `void qdump_print_stats_json(xasl_node*, json_t* parent)` | SERVER_MODE: fill JSON stats object from executed XASL |
| `void qdump_print_stats_text(FILE* fp, xasl_node*, int indent)` | SERVER_MODE: text stats output with indent level |
| `const char* qdump_operator_type_string(OPERATOR_TYPE)` | Operator enum → string (e.g., `OP_EQ` → `"eq"`) |
| `const char* qdump_default_expression_string(DB_DEFAULT_EXPR_TYPE)` | Default expression enum → string |

---

## Execution Path — EXPLAIN

```
Client: SET TRACE ON; SELECT …; SHOW TRACE;

Server side (after qexec_execute_query returns):
    if xasl_trace flag set in query:
        perfmon_start_watch / perfmon_stop_watch
        qdump_print_stats_json(xasl_p, root_json)
            qdump_print_xasl_node_stats_json(node, parent_json)
                json_object_set_new(parent, "type", type_str)
                json_object_set_new(parent, "table", table_str)
                json_object_set_new(parent, "access", access_str)
                json_object_set_new(parent, "cost", {io_cost, cpu_cost})
                json_object_set_new(parent, "rows", actual_row_count)
                for each subquery / join:
                    qdump_print_xasl_node_stats_json(sub_node, sub_json)
        json → serialize → send as query trace attribute in reply

Client: SHOW TRACE → receives JSON blob → displays
```

> [!key-insight]
> The stats JSON is assembled **post-execution** from the already-executed XASL tree. Actual row counts and I/O stats reflect what really happened, not optimizer estimates. This makes CUBRID's EXPLAIN TRACE format more like PostgreSQL's `EXPLAIN ANALYZE` than MySQL's `EXPLAIN`.

---

## PROC_TYPE Coverage

`qdump_print_xasl_type` handles all PROC_TYPE variants:

| PROC_TYPE | String |
|-----------|--------|
| BUILDLIST_PROC | `"buildlist_proc"` |
| BUILDVALUE_PROC | `"buildvalue_proc"` |
| UNION_PROC | `"union_proc"` |
| DIFFERENCE_PROC | `"difference_proc"` |
| INTERSECTION_PROC | `"intersection_proc"` |
| OBJFETCH_PROC | `"objfetch_proc"` |
| SCAN_PROC | `"scan_proc"` |
| MERGELIST_PROC | `"mergelist_proc"` |
| HASHJOIN_PROC | `"hashjoin_proc"` |
| UPDATE_PROC | `"update_proc"` |
| DELETE_PROC | `"delete_proc"` |
| INSERT_PROC | `"insert_proc"` |

---

## Key Field Decoders (Static Functions)

These cover every major XASL field and are useful for debugging deserialized plans:

| Function | Decodes |
|----------|---------|
| `qdump_print_access_spec` | `ACCESS_SPEC_TYPE`: heap scan, index scan, list scan, method scan |
| `qdump_print_key_info` | `KEY_INFO`: key ranges (BETWEEN / EQ / GT / LT …) |
| `qdump_print_index` | `INDX_INFO`: BTID, coverage, order by optimization |
| `qdump_print_predicate` | `PRED_EXPR` tree: AND/OR/NOT + comparison leaves |
| `qdump_print_arith_expression` | `ARITH_TYPE`: operator + left/right/third operands |
| `qdump_print_aggregate_expression` | `AGGREGATE_TYPE`: function name + column ref |
| `qdump_print_outlist` | `OUTPTR_LIST`: projection columns |
| `qdump_print_sort_list` | `SORT_LIST`: ORDER BY specs |
| `qdump_print_update_proc_node` | `UPDATE_PROC_NODE`: class OIDs, assignment list |
| `qdump_print_insert_proc_node` | `INSERT_PROC_NODE`: target class, value list |
| `qdump_print_hashjoin_proc_node` | `HASHJOIN_PROC_NODE`: hash method, probe/build sides |
| `qdump_print_hashjoin_stats_text/json` | Hash join execution stats (tuples, spill) |
| `qdump_print_px_subquery_stats_json` | Parallel subquery executor stats |

---

## Hash Number and Consistency Check

```c
#define HASH_NUMBER 128

// CUBRID_DEBUG only:
qdump_check_node(xasl, chk_nodes[HASH_NUMBER])
    // hash XASL node pointers, detect unreachable or duplicate nodes
qdump_print_inconsistencies(chk_nodes)
    // report any plan inconsistencies to stderr
```

The `QDUMP_XASL_CHECK_NODE` hash array tracks every XASL node pointer seen during a dump, verifying that all referenced nodes are reachable. This is a development-time plan consistency checker only (`#if defined(CUBRID_DEBUG)`).

---

## Parallel Query Stats Integration

Under `#if defined(SERVER_MODE)`, `qdump_print_px_subquery_stats_json` calls into `px_heap_scan_trace_handler` and `px_query_executor` to include parallel scan and parallel subquery execution stats in the JSON output. This requires the `parallel-query` subsystem headers.

---

## Output Formats

| Mode | Format | Destination | Trigger |
|------|--------|-------------|---------|
| `qdump_print_xasl` | Structured text | `stdout` (`foutput`) | Debug / CSQL `trace plan` text |
| `qdump_print_stats_json` | JSON (jansson) | `json_t*` → wire | CSQL `SET TRACE ON JSON` |
| `qdump_print_stats_text` | Indented text | `FILE*` | CSQL `SET TRACE ON TEXT` |

---

## Constraints

| Constraint | Detail |
|------------|--------|
| Build mode | `qdump_print_xasl`: all modes. `qdump_print_stats_json/text`: SERVER_MODE only |
| Thread safety | Read-only walk of XASL tree; no mutations; safe per-thread |
| JSON library | Uses jansson (`json_t*`); object not freed here — caller owns root |
| Memory | All string constants are static; no dynamic allocation in text paths |
| `CUBRID_DEBUG` | Consistency check code only compiled with `#if defined(CUBRID_DEBUG)` |

---

## Lifecycle

```
Post-execution (server-side, per query):
    qmgr_process_query → qexec_execute_query (execution completes)
    if QUERY_TRACE flag:
        qdump_print_stats_json(xasl_p, json_root)
        json_root → net_write_trace_attribute → client reply

Per-request (CSQL plan dump):
    CSQL: SHOW TRACE → server sends stored json_root
    CSQL: explain_print_xasl → qdump_print_xasl (SA_MODE inline)
```

---

## Related

- [[components/xasl]] — `XASL_NODE` structure decoded by all dump functions
- [[components/query-executor]] — executes the XASL; dump reads post-execution stats
- [[components/xasl-predicate]] — `PRED_EXPR` decoded by `qdump_print_predicate`
- [[components/regu-variable]] — `REGU_VARIABLE` decoded by `qdump_print_value`
- [[components/parallel-query]] — parallel stats integrated via `qdump_print_px_subquery_stats_json`
- [[components/query-manager]] — query trace flag set in query entry; dump called after execution
- [[Build Modes (SERVER SA CS)]]
