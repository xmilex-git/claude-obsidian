---
type: component
parent_module: "[[components/optimizer-rewriter]]"
path: "src/optimizer/rewriter/query_rewrite_set.c"
status: active
purpose: "Set-operation (UNION / INTERSECTION / DIFFERENCE) helpers used by the rewriter: LIMIT pushdown, DISTINCT detection, hint inheritance"
key_files:
  - "query_rewrite_set.c (153 LOC, 3 functions)"
public_api:
  - "qo_push_limit_to_union(parser, node, limit) → PT_NODE * — recursively push a LIMIT down into every SELECT arm of a union"
  - "qo_check_distinct_union(parser, node) → bool — true if any UNION node in the tree has all_distinct == PT_DISTINCT"
  - "qo_check_hint_union(parser, node, hint) → bool — true if any SELECT arm carries the given hint"
tags:
  - component
  - cubrid
  - optimizer
  - rewriter
  - union
  - limit
related:
  - "[[components/optimizer-rewriter]]"
  - "[[components/parse-tree]]"
created: 2026-04-25
updated: 2026-04-25
---

# `query_rewrite_set.c` — Set-Operation Rewrites

Tiny file (153 LOC, 3 public functions). Used exclusively by the [[components/optimizer-rewriter|rewriter orchestrator]] (`query_rewrite.c`) when handling `PT_UNION` / `PT_DIFFERENCE` / `PT_INTERSECTION` nodes that carry a `LIMIT` without `ORDER BY`. All functions recurse over the union tree, descending into the two arms (`union_.arg1`, `union_.arg2`).

## `qo_push_limit_to_union` (`:39-90`)

```c
PT_NODE *qo_push_limit_to_union (PARSER_CONTEXT *parser, PT_NODE *node, PT_NODE *limit);
```

Pushes a LIMIT clause down into every leaf `PT_SELECT` arm of a union tree. Returns the (possibly modified) `node` pointer.

Recursion:
- `PT_UNION` → recurse into both arms.
- `PT_SELECT` → if the leaf has no `INST_NUM`/`ORDERBY_NUM` in its WHERE and no existing `limit`, attach a copy of the limit:
  - Single-arg LIMIT (`limit 10`): `parser_copy_tree(limit)` is attached.
  - Two-arg LIMIT (`limit 10, 10` = offset 10, count 10): rewritten as `limit 10 + 10` (sum of offset and count) — covers all rows the outer caller could read, then the outer LIMIT slices the actual offset+count window. Implementation builds a `PT_PLUS` expression node with copies of both operands.
  - Sets `info.query.flag.rewrite_limit = 1` to mark the leaf as "limit was pushed here, may need post-processing".

Bail-out conditions on the leaf:
- `pt_has_inst_or_orderby_num_in_where (node)` returns true (the WHERE already references row-numbering pseudo-columns — pushing limit could change semantics).
- `node->info.query.limit` is already set (don't overwrite).

`PT_DIFFERENCE` and `PT_INTERSECTION` fall through to `default: break;` — limits are NOT pushed into those. Only true UNION recurses. This is correct: `(A INTERSECT B) LIMIT 10` cannot be rewritten as `(A LIMIT 10) INTERSECT (B LIMIT 10)` because the intersection might require all rows of both arms to find matches.

> [!key-insight] Two-arg LIMIT rewrite
> `LIMIT 10, 10` is converted to `LIMIT 10 + 10` (= 20) at the leaf, not `LIMIT 10` with offset preserved. The outer-level union still applies the original `LIMIT 10, 10` post-merge — the leaf-level rewrite is a **superset upper bound** that the outer can slice further. This is safe because LIMIT is a superset (returning more rows is never less correct than returning fewer).

## `qo_check_distinct_union` (`:100-121`)

```c
bool qo_check_distinct_union (PARSER_CONTEXT *parser, PT_NODE *node);
```

Recurses across the union tree and returns `true` if any `PT_UNION` node has `info.query.all_distinct == PT_DISTINCT` (i.e. the user wrote `UNION` without `ALL`). Short-circuits on first hit.

`PT_DIFFERENCE` and `PT_INTERSECTION` are not checked (they always behave as set operations regardless of the `ALL` modifier in this codebase). Pure `OR` accumulator pattern; no side effects.

Used by the orchestrator at `query_rewrite.c:192` to decide whether limit-pushdown is safe — `UNION` (distinct) requires deduplication across the full result, so pushing LIMIT to the leaves before dedup would yield wrong row counts.

## `qo_check_hint_union` (`:131-153`)

```c
bool qo_check_hint_union (PARSER_CONTEXT *parser, PT_NODE *node, PT_HINT_ENUM hint);
```

Recurses across the union tree and returns `true` if any `PT_SELECT` arm has `info.query.q.select.hint & hint`. Bitmask test — the hint is treated as a bit flag.

Used by the orchestrator at `query_rewrite.c:193` with `PT_HINT_NO_PUSH_PRED` to detect the "no predicate pushdown" hint that suppresses LIMIT pushdown across the union.

## Pre-conditions and surrounding flow

The orchestrator block at `query_rewrite.c:167-212` calls these three functions together:

```c
if (node->info.query.order_by == NULL
    && !qo_check_distinct_union (parser, node)
    && !qo_check_hint_union (parser, node, PT_HINT_NO_PUSH_PRED))
{
    node = qo_push_limit_to_union (parser, node, limit_node);
}
```

Three-way gate: no ORDER BY (else the limit is order-dependent and can't push), not a distinct union (else dedup blocks pushdown), no `NO_PUSH_PRED` hint (manual override).

## Smells / observations

- All three functions are pure CNF-style recursive predicates. No state, no mutation in the check functions, only pointer-edits in `qo_push_limit_to_union`.
- `qo_check_hint_union` only inspects `PT_SELECT` arms. Nested unions return false unless they reach a SELECT leaf with the hint — implicit assumption that hints are leaf-level.
- The `PT_PLUS` expression built for two-arg LIMIT (`limit 10 + 10`) is constant-foldable — a later rewriter pass would collapse it to `20`, but at this stage the literal-children path keeps the structure for traceability.

## Related

- Parent: [[components/optimizer-rewriter]]
- Sibling: [[components/optimizer-rewriter-select]] — handles non-set-operation LIMIT pushdown for plain `PT_SELECT`.
- [[components/parse-tree]] — `PT_HINT_ENUM`, `PT_DISTINCT`, `PT_UNION` definitions.
