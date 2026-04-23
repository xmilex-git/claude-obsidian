---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/scan_json_table.cpp + scan_json_table.hpp"
status: active
purpose: "SQL:2016 JSON_TABLE table function scanner — expands a JSON document into a relational row set using a tree of path expressions and NESTED PATH sub-nodes, implemented as a depth-first cursor walk over the JSON tree"
key_files:
  - "src/query/scan_json_table.cpp — cubscan::json_table::scanner implementation + cursor inner struct"
  - "src/query/scan_json_table.hpp — scanner class interface (init/clear/open/end/next_scan)"
  - "src/xasl/access_json_table.hpp — spec_node, node, column structs (XASL-serialised JSON_TABLE spec)"
  - "src/base/db_json.hpp — JSON_DOC, JSON_ITERATOR, db_json_extract_document_from_path, db_json_iterator_*"
public_api:
  - "scanner::init(spec) — bind to XASL spec_node; allocate cursor array; init iterators"
  - "scanner::clear(xasl_p, is_final, is_final_clear) — reset/release cursors and documents"
  - "scanner::open(thread_p) → int — fetch input JSON value, init root cursor"
  - "scanner::end(thread_p) — no-op currently"
  - "scanner::next_scan(thread_p, sid, sc) → int — generate next row or S_END"
  - "scanner::get_predicate() → SCAN_PRED& — expose scan predicate for scan_manager"
  - "scanner::set_value_descriptor(vd) — bind VAL_DESCR* for regu-variable evaluation"
tags:
  - component
  - cubrid
  - query
  - scan
  - json
  - json-table
  - sql2016
related:
  - "[[components/scan-manager|scan-manager]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/xasl|xasl]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/xasl-predicate|xasl-predicate]]"
  - "[[dependencies/rapidjson]]"
  - "[[Memory Management Conventions]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `scan_json_table.cpp` — JSON_TABLE Scanner

`cubscan::json_table::scanner` implements the `S_JSON_TABLE_SCAN` scan type for the SQL:2016 `JSON_TABLE(...)` table function. It expands an input JSON document into relational rows by walking a tree of `json_table::node` objects (root + nested paths), using a depth-first breadcrumb cursor that knows where the last row was generated.

## Purpose

`JSON_TABLE` shreds a JSON value into rows and columns using path expressions:

```sql
SELECT *
FROM   JSON_TABLE(doc, '$[*]' COLUMNS (
         rownum FOR ORDINALITY,
         a STRING PATH '$.a',
         b INT EXISTS '$.b',
         NESTED PATH '$.arr[*]' COLUMNS (c JSON PATH '$.c')
       )) AS jt
```

Each element found at the root path (`$[*]`) generates one "outer row". For each outer row, each `NESTED PATH` expands an array inside that element into additional rows. Sibling nested paths are not cross-joined — while one expands, the others emit NULL.

## Public Entry Points

| Method | Phase | Description |
|---|---|---|
| `scanner::init(spec)` | open | Compute tree height; allocate `cursor[tree_height]`; init `JSON_ITERATOR*` per node |
| `scanner::open(thread_p)` | start | `fetch_peek_dbval` the JSON regu-var; `db_value_to_json_doc` if not already JSON; `init_cursor` on root |
| `scanner::next_scan(thread_p, sid, sc)` | next | Calls `scan_next_internal` recursively; applies `m_scan_predicate`; returns `S_SUCCESS` / `S_END` / `S_ERROR` |
| `scanner::end(thread_p)` | close | Currently a no-op (resources freed in `clear`) |
| `scanner::clear(xasl_p, is_final, is_final_clear)` | reset | Clears output column `DB_VALUE*` pointers; resets ordinality; if `is_final`: clears input docs and iterators; if `is_final_clear`: `delete[] m_scan_cursor` |
| `scanner::get_predicate()` | — | Returns `m_scan_predicate` for scan_manager integration |
| `scanner::set_value_descriptor(vd)` | — | Binds `m_vd` for `fetch_peek_dbval` in `open` |

## XASL Integration

`scanner` works against `cubxasl::json_table::spec_node` (from `access_json_table.hpp`), which is serialised as part of the XASL stream client→server:

```
spec_node
  m_json_reguvar   → REGU_VARIABLE pointing to the JSON expression
  m_root_node      → json_table::node (root COLUMNS path)
    m_path         → string path for this node (e.g. "$[*]")
    m_iterator     → JSON_ITERATOR* (allocated by init_iterators)
    m_ordinality   → INT64 row counter (FOR ORDINALITY column)
    m_output_columns[]  → array of json_table::column
    m_nested_nodes[]    → child nodes for NESTED PATH
    m_is_iterable_node  → true if this node expands an array
    m_node_count        → total nodes in subtree
```

`JSON_TABLE_SCAN_ID` is a typedef alias for `cubscan::json_table::scanner`, embedded directly in `SCAN_ID` via the scan-manager union.

## Cursor Design

The scanner maintains `cursor m_scan_cursor[tree_height]`, one per depth level:

```cpp
struct scanner::cursor {
  size_t          m_child;           // which child branch is currently active
  json_table::node *m_node;          // the node at this depth
  JSON_DOC_STORE   m_input_doc;      // owned copy of the input JSON for this node
  const JSON_DOC  *m_process_doc;    // borrowed ptr (from iterator or input_doc)
  bool m_is_row_fetched;             // current row columns already evaluated
  bool m_need_advance_row;           // set after emitting a row; advance on next entry
  bool m_is_node_consumed;           // no more rows from this node at current input
  bool m_iteration_started;          // at least one child was entered for this row
};
```

> [!key-insight] Depth-first breadcrumb cursor
> `scan_next_internal(depth)` is called recursively. The `m_scan_cursor_depth` variable tracks the deepest active level. On each `next_scan` call, the recursion resumes from where it left off: leaf nodes advance their row cursor; non-leaf nodes advance to the next child when the previous child is consumed. When all children of a non-leaf row are consumed and no child was ever expanded (`m_iteration_started == false`), the non-leaf row is emitted with all child columns as NULL — this is the correct SQL:2016 behavior for rows with no matching nested path elements.

## Execution Path

```
scanner::next_scan()
  ├── [S_BEFORE] scanner::open()
  │     ├── fetch_peek_dbval(m_specp->m_json_reguvar)  ← get input JSON
  │     ├── db_value_to_json_doc() if type != DB_TYPE_JSON
  │     ├── init_cursor(root_doc, root_node, cursor[0])
  │     │     └── set_input_document → db_json_extract_document_from_path(path)
  │     │           → start_json_iterator if iterable array
  │     └── reset_ordinality(root_node)
  │
  └── [S_ON] scan_next_internal(depth=0, &has_row)
        ├── if m_scan_cursor_depth >= depth+1: recurse to child first
        ├── while !cursor.m_is_node_consumed:
        │     ├── if m_need_advance_row: advance_row_cursor()
        │     │         → db_json_iterator_next() + m_ordinality++
        │     ├── cursor.fetch_row()
        │     │     → for each output column: column.evaluate(*m_process_doc, ordinality)
        │     ├── [leaf node] found_row_output = true; return
        │     └── [non-leaf] set_next_cursor(depth+1) → recurse
        │
        └── [post-row] apply m_scan_predicate.pr_eval_fnc()
              → skip if V_FALSE; return S_SUCCESS if V_TRUE
```

## Column Extraction

Each `json_table::column` has a type:
- **PATH column**: `db_json_extract_document_from_path` extracts a value; `db_value_to_json_doc` + type cast converts to declared SQL type.
- **EXISTS column**: returns 1 if path matches, 0 otherwise.
- **FOR ORDINALITY column**: emits the `m_ordinality` counter (1-based, incremented per iterable row advance).
- **NESTED PATH**: handled by a child `json_table::node`; this column type adds no output itself.

Column evaluation calls `column.evaluate(*m_process_doc, m_ordinality)` which is defined in `access_json_table.cpp` (not in this file).

## RapidJSON Backend

JSON path extraction and iteration use the `db_json_*` abstraction layer (`src/base/db_json.hpp/cpp`), which wraps [[dependencies/rapidjson]]:
- `db_json_extract_document_from_path` — `rapidjson::Pointer` path extraction
- `db_json_set_iterator` / `db_json_iterator_next` / `db_json_iterator_get_document` — array iteration wrapper over `rapidjson::Value::ConstValueIterator`
- `db_json_get_type` — returns `DB_JSON_ARRAY`, `DB_JSON_OBJECT`, etc.
- `JSON_DOC_STORE` — RAII wrapper holding a `rapidjson::Document*`; `clear()` frees the document

> [!key-insight] Path compilation is not cached per-column
> Each `db_json_extract_document_from_path` call re-evaluates the string path at runtime (parsed by RapidJSON Pointer on each invocation). There is no pre-compiled path cache in the current implementation. For queries that scan many JSON rows, repeated path parsing is a per-row cost. The implementation comment in the header notes this as a future optimization opportunity: "We could partition the scan predicate on scan nodes and filter invalid rows at node level."

## Constraints

- **One input document per `open` call**: The JSON expression is fetched once at `open` time. If the JSON column changes between outer-table rows (i.e. `JSON_TABLE` is in a correlated context), `clear(is_final=true)` resets the cursors and `open` re-fetches on the next `next_scan` call after position resets to `S_BEFORE`.
- **No spill**: All JSON data is held in memory as `rapidjson::Document` objects. Very large JSON documents will occupy process memory for the scan duration.
- **Sibling isolation**: Sibling `NESTED PATH` nodes are not cross-joined. While one sibling is expanded, all other siblings output NULL — this is explicit SQL:2016 behavior.
- **No parallelism**: `scanner` is a single-threaded object; `SCAN_ID` and `JSON_TABLE_SCAN_ID` have no parallel state.
- **Thread safety**: Not thread-safe. Each query execution creates its own `scanner` instance inside `SCAN_ID`.
- **C++17**: `scanner` uses `std::vector` (via `#include <algorithm>`), `new[]`/`delete[]` for cursor array. Memory is **not** `db_private_alloc`; cursor array is `new cursor[tree_height]` and freed in `clear(is_final_clear=true)`.

## Lifecycle

```
init          : scanner::init(spec) — compute tree_height; new cursor[]; init_iterators
open          : scanner::open(thread_p) — fetch JSON reguvar; init root cursor with input doc
next [loop]   : scanner::next_scan() — scan_next_internal DFS; predicate filter; emit one row
clear(partial): scanner::clear(is_final=false) — clear column output values; reset ordinality
clear(final)  : scanner::clear(is_final=true, is_final_clear=true) — clear all docs, iterators, delete[]
end           : scanner::end() — no-op
```

When `JSON_TABLE` is used inside a correlated subquery or alongside a driving outer table, `scan_manager` calls `clear(is_final=true)` + resets `position = S_BEFORE` before each outer row. `next_scan` detects `S_BEFORE` and re-calls `open`, re-fetching the JSON expression for the new outer row.

## Related

- [[components/scan-manager|scan-manager]] — `S_JSON_TABLE_SCAN` dispatch; `JSON_TABLE_SCAN_ID` = typedef of `scanner`
- [[components/query-executor|query-executor]] — generates `S_JSON_TABLE_SCAN` access specs for `JSON_TABLE(...)` in FROM clause
- [[components/xasl|xasl]] — `cubxasl::json_table::spec_node` / `node` / `column` are serialised as part of the XASL stream
- [[components/regu-variable|regu-variable]] — `m_json_reguvar` is a `REGU_VARIABLE*` providing the JSON input expression
- [[components/xasl-predicate|xasl-predicate]] — `m_scan_predicate` is a `SCAN_PRED` with `PRED_EXPR*` for WHERE filtering
- [[dependencies/rapidjson]] — underlying JSON DOM and path extraction engine
- [[Memory Management Conventions]] — cursor array uses `new[]`, not `db_private_alloc`; a deliberate exception noted in the code
