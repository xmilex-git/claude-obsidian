---
type: component
parent_module: "[[components/optimizer-rewriter]]"
path: "src/optimizer/rewriter/query_rewrite_auto_parameterize.c"
status: active
purpose: "Replace literal constants with input host-variable markers for plan cache reuse: WHERE predicates, LIMIT/OFFSET clauses, KEYLIMIT clauses"
key_files:
  - "query_rewrite_auto_parameterize.c (361 LOC, 3 public functions)"
public_api:
  - "qo_auto_parameterize(parser, where) — convert constants in WHERE-shaped CNF/DNF list to host-var input markers"
  - "qo_auto_parameterize_limit_clause(parser, node) — parameterize LIMIT (and offset, when 2-arg)"
  - "qo_auto_parameterize_keylimit_clause(parser, node) — parameterize indx_key_limit on USING-INDEX names"
tags:
  - component
  - cubrid
  - optimizer
  - rewriter
  - auto-parameterize
  - xasl-cache
related:
  - "[[components/optimizer-rewriter]]"
  - "[[components/xasl-cache]]"
  - "[[components/parser]]"
created: 2026-04-25
updated: 2026-04-25
---

# `query_rewrite_auto_parameterize.c` — Constant → Host Variable

Three functions, ~360 LOC. Performs the **last** step of the rewriter pipeline: turning literal constants in supported syntactic positions into host-variable input markers (`?`) so the resulting XASL plan is cacheable across query texts that differ only in literal values.

The actual `pt_rewrite_to_auto_param` worker — which allocates a new host-var index, copies the constant into the parameter array, and replaces the `PT_VALUE` node with a `PT_HOST_VAR` input marker — lives in the parser layer (`parser/`). This file owns the **dispatch** logic: walking the relevant slots and deciding which constants are eligible.

## When auto-parameterization runs

From [[components/optimizer-rewriter]] orchestrator (`query_rewrite.c:494-501`):

```c
if (!prm_get_bool_value (PRM_ID_HOSTVAR_LATE_BINDING)
    && prm_get_integer_value (PRM_ID_XASL_CACHE_MAX_ENTRIES) > 0
    && node->flag.cannot_prepare == 0
    && parser->flag.is_parsing_static_sql == 0
    && parser->flag.is_skip_auto_parameterize == 0)
{
    call_auto_parameterize = true;
}
```

Five-way gate: late-binding off, XASL cache enabled, statement is preparable, not a static-SQL parse, and not a skip-auto-parameterize parser flag.

`qo_auto_parameterize_limit_clause` and `qo_auto_parameterize_keylimit_clause` are called **unconditionally** later (`query_rewrite.c:550-560`) — they have their own internal skip-flag checks.

## `qo_auto_parameterize` — WHERE-shape walker (`:40-126`)

```c
void qo_auto_parameterize (PARSER_CONTEXT *parser, PT_NODE *where);
```

Walks a CNF list (terms linked by `next`), then the DNF list inside each CNF term (linked by `or_next`). For each `PT_EXPR` DNF term:

### Pre-checks

1. Skip if `PT_EXPR_INFO_DO_NOT_AUTOPARAM` flag is set (`:59-65`). Comment: "copy_pull term from select list of derived table do NOT auto_parameterize because the query rewrite step is performed in the XASL generation of DELETE and UPDATE." TODO comment notes the desire to remove this constraint.
2. Compute `node_prior = pt_get_first_arg_ignore_prior(dnf_node)` — gets `arg1` past any wrapping `PT_PRIOR` (CONNECT BY pseudo-column).
3. Skip unless `node_prior` is one of: `pt_is_attr`, `pt_is_function_index_expression`, `pt_is_instnum`, `pt_is_orderbynum`. Auto-parameterization only fires when LHS is a column reference or row-number pseudo-column.

### Per-operator parameterization

| Operator | Slot(s) auto-parameterized |
|---|---|
| `PT_EQ`, `PT_GT`, `PT_GE`, `PT_LT`, `PT_LE`, `PT_LIKE`, `PT_ASSIGN` | `arg2` if `pt_is_const_not_hostvar` and not NULL |
| `PT_BETWEEN` | `between_and->arg1` and `between_and->arg2` (the two boundaries) |
| `PT_RANGE` | for each `range` in the OR-chain: both `arg1` and `arg2` of each range, but **NOT collection-typed** values |
| anything else | skip |

`PT_NE` is NOT in the list — comment hints at "Is any other expression type possible to be auto-parameterized?" (open question).

### NULL handling

`PT_IS_NULL_NODE(value)` short-circuits parameterization for explicit NULL literals. NULL stays a literal because parameter binding can't carry a typed NULL.

### Collection types in PT_RANGE

For `PT_RANGE`, `PT_IS_COLLECTION_TYPE(arg->type_enum)` blocks parameterization. Set/multiset/sequence values can't be auto-parameterized as scalar host vars; they would need a structured parameter representation that doesn't exist.

## `qo_auto_parameterize_limit_clause` — LIMIT / OFFSET (`:128-290`)

```c
void qo_auto_parameterize_limit_clause (PARSER_CONTEXT *parser, PT_NODE *node);
```

Handles SELECT/UNION/UPDATE/DELETE LIMIT clauses. Two-arg LIMIT is split into `(offset, count)` via `limit->next` linking.

### Internal skip flags

`parser->flag.is_parsing_static_sql == 1 || parser->flag.is_skip_auto_parameterize == 1` → return early. Note this duplicates the orchestrator-level skip but the function is also called unconditionally from `query_rewrite.c:553`, so the duplication is necessary.

### Per-statement-type slot binding

| Statement | Slot |
|---|---|
| `PT_UNION` / `PT_DIFFERENCE` / `PT_INTERSECTION` / `PT_SELECT` | `info.query.limit` |
| `PT_UPDATE` | `info.update.limit` |
| `PT_DELETE` | `info.delete_.limit` |

Other types → return.

### LIMIT structure

If `limit->next != NULL`: 2-arg form (`LIMIT offset, count`). `limit_offsetp = limit`, `limit_row_countp = limit->next`. The `next` pointer is **cut** during processing.

If `limit->next == NULL`: 1-arg form (`LIMIT count`). `limit_offsetp = NULL`, `limit_row_countp = limit`.

### Parameterization

Both `limit_offsetp` and `limit_row_countp` are individually checked:
- `pt_is_const_not_hostvar(node) && !PT_IS_NULL_NODE(node)` → call `pt_rewrite_to_auto_param`.
- Otherwise: leave alone.

A commented-out `#if 0` block at `:220-229` and `:239-248` notes that arbitrary expressions in LIMIT (e.g. `LIMIT 0 + 2`) could in principle be parameterized but the author judged it not practical: "Full constant expressions, e.g, (0+2) is folded as constant and eventually parameterized as a hostvar. Expressions which include a const would be mixed use of a constant and a hostvar, e.g, (0+?). If you really want to optimize this case too, you can add a function to parameterize an expression node."

After parameterization, the offset and row-count are re-linked back into the appropriate `node->info.update.limit` / `node->info.delete_.limit` / `node->info.query.limit` slot.

## `qo_auto_parameterize_keylimit_clause` — index KEYLIMIT (`:292-361`)

```c
void qo_auto_parameterize_keylimit_clause (PARSER_CONTEXT *parser, PT_NODE *node);
```

Walks `using_index` clause names (`USING INDEX i KEYLIMIT a, b`). For each name node, the `info.name.indx_key_limit` linked list carries the upper bound; `indx_key_limit->next` is the lower bound.

Same `pt_is_const_not_hostvar` test parameterizes each bound independently.

Per-statement source:

| Statement | Source |
|---|---|
| `PT_SELECT` | `info.query.q.select.using_index` |
| `PT_UPDATE` | `info.update.using_index` |
| `PT_DELETE` | `info.delete_.using_index` |

### Internal handling

The `next` pointer is cut at `:332` (`indx_key_limit->next = NULL`) before parameterizing the upper bound, then the lower bound is parameterized and re-linked at `:351`.

If the bound is already a host var or NULL, the original node is preserved.

## Smells / observations

- The TODO comment on `PT_EXPR_INFO_DO_NOT_AUTOPARAM` (`:62-63`) flags a known wart: copy-pulled terms from derived-table select lists need to be re-parameterized at XASL generation time, breaking the rule that the rewriter is supposed to be the last touch.
- The "Is any other expression type possible?" comment at `:120` is an open invitation to extend the operator list — `PT_NE` is the most obvious candidate (and it's commonly auto-parameterized in other DBs).
- The `#if 0` blocks for expression-level parameterization are dead code that documents an explicit non-goal. Either remove or wrap in a documented `#if 0` discussion comment.
- `qo_auto_parameterize` does NOT touch `PT_IN` lists. `WHERE x IN (1, 2, 3)` keeps the literals; only `PT_EQ` / range comparisons get parameterized. This is presumably to preserve list-cardinality information for the planner; consequence is that `IN`-heavy workloads see fewer plan cache hits.
- The function is structured around CNF×DNF traversal but only mutates DNF terms; CNF traversal order does not matter (no inter-term state).

## Cross-references

- Calls: `pt_get_first_arg_ignore_prior`, `pt_is_attr`, `pt_is_function_index_expression`, `pt_is_instnum`, `pt_is_orderbynum`, `pt_is_const_not_hostvar`, `pt_rewrite_to_auto_param`.
- Used flags: `PT_EXPR_INFO_DO_NOT_AUTOPARAM`, `PT_IS_NULL_NODE`, `PT_IS_COLLECTION_TYPE`, `PT_EXPR_INFO_IS_FLAGED`.
- Related sysprm: `PRM_ID_HOSTVAR_LATE_BINDING`, `PRM_ID_XASL_CACHE_MAX_ENTRIES` (gate the orchestrator's call into here).

## Related

- Parent: [[components/optimizer-rewriter]]
- [[components/xasl-cache]] — auto-parameterized plans key into the XASL cache for reuse across literal-only-different statements.
