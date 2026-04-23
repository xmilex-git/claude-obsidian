---
type: component
parent_module: "[[components/parser|parser]]"
path: "src/parser/name_resolution.c"
status: active
purpose: "Bind every PT_NAME identifier to its schema object (DB_OBJECT*) and resolve PT_SPEC flat-entity lists; implements lexical scope stack for correlated subqueries"
key_files:
  - "name_resolution.c (main implementation — ~5000 lines)"
  - "parser.h (pt_resolve_default_value, pt_bind_values_to_hostvars)"
  - "semantic_check.h (SEMANTIC_CHK_INFO passed through)"
public_api:
  - "pt_bind_names(parser, node, arg, continue_walk) — walk callback; primary entry point"
  - "pt_resolve_names(parser, statement, sc_info) — top-level driver (called from compile.c)"
  - "pt_bind_values_to_hostvars(parser, node) — bind DB_VALUE array to PT_HOST_VAR nodes"
  - "pt_make_flat_name_list(parser, spec, spec_parent, for_update) — expand class hierarchy"
tags:
  - component
  - cubrid
  - parser
  - name-resolution
related:
  - "[[components/parser|parser]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/semantic-check|semantic-check]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# Name Resolution (`name_resolution.c`)

Name resolution walks the `PT_NODE` tree and binds every bare identifier (`PT_NAME`) to a concrete schema object (`DB_OBJECT *`) or a position in the `FROM`-clause scope. It is the first semantic pass after parsing.

## Scope model

```c
typedef struct scopes SCOPES;
struct scopes {
  SCOPES    *next;              // enclosing scope (linked list = scope chain)
  PT_NODE   *specs;             // list of PT_SPEC nodes visible at this level
  unsigned short correlation_level; // how many levels up was a name found?
  short     location;           // outer-join location tracking
};
```

Each `SELECT` statement pushes a new `SCOPES` frame. Correlated subquery references increment `correlation_level` in the owning `PT_NODE.info.query.correlation_level`.

`PT_BIND_NAMES_ARG` bundles the current scope chain, an `extra_specs_frame` list (for outer-join look-aside), and a `SEMANTIC_CHK_INFO *` passed through from `compile.c`.

## Resolution algorithm

`pt_bind_names` is a `parser_walk_tree` pre-order callback. For each node:

1. **`PT_SPEC`** — call `pt_make_flat_name_list` to expand the class/view hierarchy into `spec.flat_entity_list`, push the spec onto the current scope.
2. **`PT_SELECT`** — push a new `SCOPES` frame onto the scope stack before visiting children; pop after.
3. **`PT_NAME`** — call `pt_get_resolution` to walk the scope chain:
   - Try `pt_find_name_in_spec` against each visible `PT_SPEC`.
   - If found: set `name.spec_id`, `name.resolved`, `name.db_object`.
   - If not found at current level: try enclosing scopes; set `correlation_level`.
   - If nowhere: report `ER_PT_SEMANTIC` error.
4. **`PT_HOST_VAR`** — call `pt_bind_type_of_host_var` to propagate the expected domain.
5. **`PT_FUNCTION`** (generic) — call `pt_find_function_type` to resolve to a `FUNC_CODE`.
6. **`PT_METHOD_CALL`** — call `pt_resolve_method` to look up the method in `sm_find_class`.

`pt_bind_names_post` is the post-order callback; it handles NATURAL JOIN attribute merging and Oracle-style `(+)` outer-join annotation.

## Class hierarchy flattening

```c
static PT_NODE *pt_make_flat_name_list(
    PARSER_CONTEXT *parser, PT_NODE *spec,
    PT_NODE *spec_parent, bool for_update);
```

For a real table spec, walks the `sm_*` schema API to collect all sub-classes (using `pt_make_subclass_list`) and stores them in `spec.flat_entity_list`. The `MHT_TABLE *names_mht` hash table detects cycles. This is what allows `SELECT * FROM person` to also scan `student` and `employee` rows when inheritance is used.

## Oracle-style outer join

`pt_check_Oracle_outerjoin` detects `col(+) = col` syntax and converts it to standard `LEFT OUTER JOIN` by flagging `PT_SPEC`. The scope `location` field tracks which side of the join each spec is on.

## SHOW statement attributes

`pt_get_all_showstmt_attributes_and_types` generates synthetic `PT_NAME`/`PT_DATA_TYPE` nodes for `SHOW` statement columns by consulting `showstmt_get_attributes(show_type)` from `show_meta.c`.

## Hint resolution

`pt_resolve_hint` walks `info.query.hint` argument lists (table names referenced in `USE_NL`, `USE_IDX`, etc.) and resolves them against the `FROM` spec list. Unresolvable hint arguments are silently discarded (non-fatal) unless `discard_no_match == REQUIRE_ALL_MATCH`.

## Key internal helpers

| Function | Purpose |
|----------|---------|
| `pt_find_name_in_spec` | Check if a name matches any column of a single PT_SPEC |
| `pt_find_attr_in_class_list` | Check flat_entity_list for an attribute |
| `pt_resolve_correlation` | Look up a name as an exposed alias (correlation name) |
| `pt_expand_external_path` | Expand path expressions (`a.b.c`) into PT_DOT_ chains |
| `pt_get_all_attributes_and_types` | Build full attribute list for a PT_SPEC |
| `pt_object_to_data_type` | Convert `DB_OBJECT *` to `PT_DATA_TYPE` node |
| `pt_make_method_call` | Synthesise a `PT_METHOD_CALL` node from a generic function call |

## Interaction with semantic_check

Name resolution sets `PT_NAME.db_object` and `PT_SPEC.flat_entity_list`. Semantic check (`pt_semantic_check`) later validates that these are used correctly (e.g. update targets are updatable, column references are unambiguous). The two passes share `SEMANTIC_CHK_INFO`.

## Invariants after this pass

- Every `PT_NAME` with `meta_class == PT_NORMAL` has a non-NULL `db_object` or `spec_id`.
- Every `PT_SPEC` with `entity_name != NULL` has a populated `flat_entity_list`.
- `PT_SPEC.id` and all `PT_NAME.spec_id` values that reference it are equal.
- Correlated subquery nodes have `info.query.correlation_level > 0`.

## Related

- Parent: [[components/parser|parser]]
- [[components/parse-tree|parse-tree]] — PT_NAME and PT_SPEC structures
- [[components/semantic-check|semantic-check]] — next pass after name resolution
- [[components/show-meta|show-meta]] — provides attribute metadata for SHOW statements
