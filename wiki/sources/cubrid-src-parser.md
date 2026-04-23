---
type: source
title: "CUBRID src/parser/ — SQL Parsing & Analysis"
source_path: "src/parser/"
ingested: 2026-04-23
tags:
  - source
  - cubrid
  - parser
  - sql
related:
  - "[[components/parser|parser]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/name-resolution|name-resolution]]"
  - "[[components/semantic-check|semantic-check]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[components/view-transform|view-transform]]"
  - "[[components/parser-allocator|parser-allocator]]"
  - "[[components/show-meta|show-meta]]"
  - "[[Query Processing Pipeline]]"
---

# Source: `src/parser/` — CUBRID SQL Parser

## Files read

| File | Method | Purpose |
|------|--------|---------|
| `AGENTS.md` | Full read | Project guide — pipeline, key files, conventions |
| `parse_tree.h` | Sampled (~4000 lines) | PT_NODE struct, enums, macros — central data model |
| `parser.h` | Full read | Public API surface |
| `semantic_check.h` | Full read | Semantic check public API |
| `parse_type.hpp` | Full read | pt_arg_type, pt_generic_type_enum |
| `parser_allocator.hpp` | Full read | C++ block_allocator wrapper |
| `xasl_generation.h` | Full read (~150 lines) | SYMBOL_INFO, TABLE_INFO, translation structs |
| `view_transform.h` | Full read | mq_translate and friends |
| `show_meta.h` | Full read | SHOWSTMT_METADATA struct |
| `csql_grammar.y` | Head (200 lines) | Preamble, container types, global state vars |
| `name_resolution.c` | Head (180 lines) | SCOPES, internal typedefs, static function list |
| `xasl_generation.c` | Head (100 lines) | Include list, HASHABLE/HASH_ATTR |

## Key findings

### 1. PT_NODE is a 30-field monolithic struct with 25 packed flag bits
The actual `struct parser_node` definition (line 3649 of `parse_tree.h`) shows considerably more fields than typically documented — including `spec_ident`, `xasl_id`, `expr_before_const_folding`, `sub_host_var_count`, and `cache_time`. The flags bitfield has ~25 members including `do_not_fold`, `is_system_generated_stmt`, and `print_in_value_for_dblink`.

### 2. PT_NODE_TYPE is function-table indexed — order is sacred
Comment at line 876 of `parse_tree.h` warns: function tables for `parser_new_node`, `parser_init_node`, `parser_print_tree` are indexed by `PT_NODE_TYPE` ordinal. New node types must have their table entries added in exact enum order.

### 3. The bison grammar has 1,000,000 stack depth and uses container_N structs
`YYMAXDEPTH 1000000` is set at the top of `csql_grammar.y`. The preamble defines `container_2` through `container_11` (groups of `PT_NODE *` fields) to pass multiple sub-trees as a single Bison `$$` value up rule stacks.

### 4. Name resolution uses a SCOPES linked-list stack with correlation level tracking
`name_resolution.c` defines a `SCOPES` struct (line 78) and `PT_BIND_NAMES_ARG`. Correlated subquery depth is tracked via `SCOPES.correlation_level`. A hash table (`PT_NAMES_HASH_SIZE = 50`) is used for fast attribute lookup within a spec.

### 5. XASL generation has its own scope stack (SYMBOL_INFO / TABLE_INFO)
Separate from the name-resolution scope stack. `SYMBOL_INFO` maintains `table_info` linked lists with `attribute_list` and `value_list` entries per table, linking parse-tree names to runtime `DB_VALUE` slots.

### 6. parser_block_allocator.dealloc is a no-op — arena lifetime
`parser_allocator.hpp` confirms `dealloc` has no implementation. The arena is freed wholesale at `parser_free_parser`. C++ objects allocated there must be trivially destructible.

## Pages created from this source

- [[components/parser|parser]] — upgraded from stub to comprehensive page
- [[components/parse-tree|parse-tree]] — PT_NODE deep dive
- [[components/name-resolution|name-resolution]] — SCOPES algorithm
- [[components/semantic-check|semantic-check]] — semantic_check + type_checking
- [[components/xasl-generation|xasl-generation]] — PT_NODE → XASL_NODE
- [[components/view-transform|view-transform]] — mq_translate, updatability
- [[components/parser-allocator|parser-allocator]] — arena lifetime model
- [[components/show-meta|show-meta]] — SHOW statement registry
