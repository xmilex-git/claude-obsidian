---
type: component
parent_module: "[[components/optimizer-rewriter]]"
path: "src/optimizer/rewriter/query_rewrite_unused_function.c"
status: dead-code
purpose: "Three legacy path-conversion / path-as-join helpers, gated entirely behind ENABLE_UNUSED_FUNCTION — never compiled into release builds; preserved as historical reference"
key_files:
  - "query_rewrite_unused_function.c (201 LOC, all #if defined(ENABLE_UNUSED_FUNCTION))"
public_api: []
tags:
  - component
  - cubrid
  - optimizer
  - rewriter
  - dead-code
related:
  - "[[components/optimizer-rewriter]]"
  - "[[components/parse-tree]]"
created: 2026-04-25
updated: 2026-04-25
---

# `query_rewrite_unused_function.c` — Disabled Legacy Helpers

Entire file content is wrapped in `#if defined(ENABLE_UNUSED_FUNCTION)` (line 23). `ENABLE_UNUSED_FUNCTION` is **never** defined in the build configuration — these functions are dead code retained as historical reference. The file's outer comment block uses the misnomer `query_rewrite_backup.c` (line 20) — a stale rename artefact.

The wiki documents this file because future ingest may surface other `#if defined(ENABLE_UNUSED_FUNCTION)` blocks elsewhere in CUBRID, and the convention here (preserve unused code, don't delete) is one this codebase uses.

## Functions (all dead)

### `qo_is_partition_attr` (`:30-48`)

Predicate: returns `1` if a `PT_NAME` node references a partition attribute (after `pt_get_end_path_node` normalizes path expressions to their terminal name). Test:

```c
node->node_type == PT_NAME
&& node->info.name.meta_class == PT_NORMAL
&& node->info.name.spec_id  /* non-zero */
&& node->info.name.partition  /* non-NULL */
```

Purpose: distinguish partition-key columns from regular columns during a path-related rewrite. The partition check today happens elsewhere in the optimizer; this helper was an earlier formulation.

### `qo_convert_path_to_name` (`:58-78`)

Replaces a `PT_DOT_` (path-expression) node with its `arg2` `PT_NAME` child if the name's `spec_id` matches the supplied spec. Used as a `parser_walk_tree` callback during path-to-join rewriting:

```
spec.path_attr  →  path_attr   (when spec.id == path_attr.spec_id)
```

Carries the previous node's `next` pointer onto the replaced name and frees the discarded `PT_DOT_` envelope. Sets `name->info.name.resolved` to the spec's range_var name.

### `qo_rewrite_as_join` (`:89-104`)

Given a query, a root spec, and a path-spec pointer, splices the path-spec into the FROM list (right after the root) and moves the path's `path_conjuncts` (the implicit join condition) into the WHERE clause. Then walks the statement to convert any `PT_DOT_` references to plain `PT_NAME` references via `qo_convert_path_to_name`.

This is the "rewrite path expression as inner join" transformation — `SELECT a.b.c FROM x a` becomes `SELECT b.c FROM x a, x_b b WHERE a.fk = b.pk` (path-expression unrolled into a join).

### `qo_rewrite_as_derived` (`:117-201`)

Heavier variant: rewrites a path spec as a **derived table** that joins the root spec with the path spec. Used when the path traversal is deep enough or in a context (likely outer paths) where a plain inner-join rewrite would lose semantics.

Steps (high level):
1. Copy the path spec to `new_spec`; detach `path_conjuncts`.
2. If the root is itself a derived spec, copy that spec's underlying query as `query`; otherwise build a fresh `PT_SELECT` over a copy of the root.
3. Append `new_spec` into the new query's FROM list.
4. Set `query->info.query.all_distinct = PT_DISTINCT` (path expressions don't preserve duplicates here).
5. Move the root WHERE plus the path conjuncts into the new query's WHERE.
6. Build a select list from `path_spec->info.spec.referenced_attrs` (qualified to `new_spec->range_var`).
7. Run `mq_regenerate_if_ambiguous` and two `mq_set_references` passes to fix up name resolution.
8. Build `path_spec->info.spec.as_attr_list` for positional correspondence with the derived select list.
9. Finally, replace the original `path_spec`'s `entity_name` and `flat_entity_list` with `derived_table = query` (so it presents as a derived spec to downstream code).

## Why these are dead

CUBRID's modern path-handling lives in `query_rewrite_select.c` under `qo_analyze_path_join_pre` / `qo_analyze_path_join` and the `view_transform.c` machinery. The two `qo_rewrite_as_*` functions here represent an earlier approach that was superseded but not deleted.

## Smells / observations

- File header comment says `query_rewrite_backup.c` — copy-paste from a renaming pass.
- The `#if defined(ENABLE_UNUSED_FUNCTION)` wrap is at the file scope (covers everything except the license header). Removing the file would have the same effect.
- No public header entries — these are all `static` so they wouldn't be linker-visible even if compiled.
- Useful as a reference for understanding the historical "path expression as join" rewrite pattern, which is what the modern `qo_analyze_path_join` family ultimately implements with different machinery.

## Related

- Parent: [[components/optimizer-rewriter]]
- Modern equivalent: `qo_analyze_path_join_pre` / `qo_analyze_path_join` in [[components/optimizer-rewriter-select]].
- [[components/parse-tree]] — `PT_DOT_`, `PT_NAME`, `PT_SPEC` definitions.
