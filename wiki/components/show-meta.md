---
type: component
parent_module: "[[components/parser|parser]]"
path: "src/parser/show_meta.c, src/parser/show_meta.h"
status: active
purpose: "Registry of SHOW statement metadata: column definitions, ORDER BY spec, argument rules, DBA-only flag, and semantic-check hook"
key_files:
  - "show_meta.h (SHOWSTMT_METADATA, SHOWSTMT_COLUMN, SHOWSTMT_NAMED_ARG structs; showstmt_get_metadata)"
  - "show_meta.c (static metadata tables, showstmt_metadata_init/final)"
public_api:
  - "showstmt_metadata_init() → int"
  - "showstmt_metadata_final()"
  - "showstmt_get_metadata(SHOWSTMT_TYPE) → const SHOWSTMT_METADATA*"
  - "showstmt_get_attributes(SHOWSTMT_TYPE) → DB_ATTRIBUTE*"
tags:
  - component
  - cubrid
  - parser
  - show
related:
  - "[[components/parser|parser]]"
  - "[[components/name-resolution|name-resolution]]"
  - "[[components/semantic-check|semantic-check]]"
created: 2026-04-23
updated: 2026-04-23
---

# SHOW Statement Metadata (`show_meta.c`)

A static registry that describes every `SHOW` statement variant (e.g. `SHOW TABLES`, `SHOW INDEXES`, `SHOW COLUMNS`, `SHOW STATUS`) — its output columns, ordering, named arguments, DBA restriction, and an optional semantic check callback.

## Key structs (`show_meta.h`)

```c
// One output column of a SHOW statement
struct showstmt_column {
  const char *name;   // column name
  const char *type;   // SQL type string (e.g. "VARCHAR(255)")
};

// ORDER BY specification for the output
struct showstmt_column_orderby {
  int  pos;   // 1-based column position
  bool asc;   // true = ASC
};

// A named argument in the SHOW syntax (e.g. "SHOW INDEXES FROM <table>")
struct showstmt_named_arg {
  const char   *name;      // argument keyword (used in syntax)
  ARG_VALUE_TYPE type;     // AVT_INTEGER | AVT_STRING | AVT_IDENTIFIER
  bool          optional;  // may be omitted
};

// Full metadata for one SHOW variant
struct showstmt_metadata {
  SHOWSTMT_TYPE          show_type;
  bool                   only_for_dba;         // DBA-only access check
  const char            *alias_print;           // display name for pt_print_select
  const SHOWSTMT_COLUMN *cols;                  // column definitions
  int                    num_cols;
  const SHOWSTMT_COLUMN_ORDERBY *orderby;       // default sort
  int                    num_orderby;
  const SHOWSTMT_NAMED_ARG *args;               // argument rules
  int                    arg_size;
  SHOW_SEMANTIC_CHECK_FUNC semantic_check_func; // optional per-type check callback
  DB_ATTRIBUTE          *showstmt_attrs;        // synthesised DB_ATTRIBUTE list
};
```

`SHOW_SEMANTIC_CHECK_FUNC` is `typedef PT_NODE *(*)(PARSER_CONTEXT *parser, PT_NODE *node)` — called from `semantic_check.c` to validate the specific arguments of each SHOW type.

## Lifecycle

```c
showstmt_metadata_init();     // builds DB_ATTRIBUTE lists; called at parser init
// ... parse/execute SHOW statements ...
showstmt_metadata_final();    // frees DB_ATTRIBUTE lists; called at parser teardown
```

`showstmt_metadata_init` iterates over all registered `SHOWSTMT_METADATA` entries and synthesises `DB_ATTRIBUTE *` linked lists from the `cols` array. These attribute lists are consumed by [[components/name-resolution|name_resolution]] via `pt_get_all_showstmt_attributes_and_types` to generate the correct `PT_NAME`/`PT_DATA_TYPE` nodes for `SHOW` statement columns.

## Integration with the parser pipeline

1. **Grammar** (`csql_grammar.y`) — recognises `SHOW <keyword>` and creates a `PT_SHOWSTMT` node with `show_type` set.
2. **Name resolution** (`name_resolution.c`) — calls `showstmt_get_attributes(show_type)` to get the synthetic column list, then builds `PT_NAME` nodes for each column.
3. **Semantic check** (`semantic_check.c`) — calls `meta->semantic_check_func(parser, node)` if non-NULL to validate show-specific arguments (e.g. verifying a table name exists for `SHOW INDEXES FROM <table>`).
4. **XASL generation** (`xasl_generation.c`) — treats `PT_SHOWSTMT` as a special scan that maps to a server-side catalog query.

## DBA guard

`only_for_dba == true` means the statement must be rejected for non-DBA users. This check is applied in semantic_check before the statement proceeds further.

## ARG_VALUE_TYPE

```c
typedef enum {
  AVT_INTEGER,    // e.g. SHOW STATUS n
  AVT_STRING,     // e.g. SHOW STATUS 'prefix'
  AVT_IDENTIFIER  // e.g. SHOW INDEXES FROM tablename
} ARG_VALUE_TYPE;
```

Argument value types are validated in the per-type `semantic_check_func`.

## Related

- Parent: [[components/parser|parser]]
- [[components/name-resolution|name-resolution]] — consumes `showstmt_get_attributes` to build column PT_NAMEs
- [[components/semantic-check|semantic-check]] — calls `semantic_check_func` hook
