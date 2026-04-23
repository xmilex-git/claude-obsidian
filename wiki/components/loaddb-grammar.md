---
type: component
parent_module: "[[modules/src|src]]"
path: "src/loaddb/"
status: active
purpose: "Bison LALR(1) grammar and flex C++ lexer for CUBRID's loaddb input file format — a distinct mini-language, separate from the SQL grammar"
key_files:
  - "load_grammar.yy (bison grammar, cubload namespace, LALR(1) C++ parser)"
  - "load_lexer.l (flex C++ scanner, cubload::scanner class)"
  - "load_scanner.hpp (scanner class declaration)"
  - "load_common.hpp (constant_type, string_type, class_command_spec_type, object_ref_type)"
tags:
  - component
  - cubrid
  - loaddb
  - bison
  - flex
  - grammar
related:
  - "[[components/loaddb|loaddb]]"
  - "[[components/loaddb-driver|loaddb-driver]]"
  - "[[components/parser|parser]]"
created: 2026-04-23
updated: 2026-04-23
---

# loaddb Grammar & Lexer

The loaddb input format is parsed by its own LALR(1) bison grammar (`load_grammar.yy`) and flex C++ lexer (`load_lexer.l`). This is an entirely separate grammar from the main SQL grammar (`csql_grammar.y`) — it describes a data-dump format, not SQL syntax.

Hub: [[components/loaddb|loaddb]].

## Grammar technology

```
%skeleton "lalr1.cc"     -- C++ LALR(1) skeleton (modern bison ≥3.0)
%require "3.0"
%defines                 -- emit load_grammar.hpp header
%define api.namespace { cubload }
%define parser_class_name { parser }
%locations               -- track source location (line/column)
%no-lines                -- suppress #line directives
%parse-param { driver &m_driver }
```

The grammar produces `cubload::parser`; the lexer is `cubload::scanner` (declared in `load_scanner.hpp`). The driver wires them together:

```cpp
// load_driver.cpp
int driver::parse(std::istream &iss, int line_offset)
{
  m_scanner->switch_streams(&iss);
  m_scanner->set_lineno(line_offset + 1);
  m_semantic_helper.reset_after_batch();
  cubload::parser parser(*this);
  return parser.parse();   // 0 = success (bison convention)
}
```

Unlike `csql_grammar.y` (which uses the legacy C Bison skeleton), this grammar uses the modern C++ skeleton — no global yylval, no yyparse(), all state in the parser/scanner objects.

## Lexer options

```
%option c++         -- C++ scanner class
%option batch       -- "somewhat more optimized" mode
```

The lexer uses `BEGIN_SUPPRESS_WARNING_BISON_FLEX` / `END_SUPPRESS_WARNING_BISON_FLEX` pragmas to silence generated-code warnings (`-Wsign-compare`, `-Wregister`, `-Wimplicit-fallthrough`).

Debug mode: set `CUBRID_LOADER_DEBUG=1` environment variable; grammar sets `yydebug` via `loader_init_yydebug()`.

## Grammar start symbol and top-level structure

```
loader_start
  └── loader_lines
        └── line*
              └── one_line
                    ├── command_line   (%class / %id directives)
                    └── instance_line  (data rows)
```

At the end of `loader_start`, the grammar action calls:
```cpp
m_driver.get_object_loader().flush_records();
m_driver.get_object_loader().destroy();
```

## Command lines

### `%id` directive

```
CMD_ID IDENTIFIER DOT IDENTIFIER INT_LIT   -- schema.table classnum
CMD_ID IDENTIFIER INT_LIT                  -- table classnum
```

Calls `class_installer::check_class(name, id)`. Used to number each class in multi-class dump files.

### `%class` directive

```
CMD_CLASS [IDENTIFIER DOT] IDENTIFIER class_command_spec
```

Where `class_command_spec` is:
```
attribute_list                              -- default attrs
attribute_list_type attribute_list          -- CLASS / SHARED / DEFAULT modifier
attribute_list constructor_spec            -- with constructor
```

`attribute_list_type` is one of: `CLASS`, `SHARED`, `DEFAULT` — mapped to `LDR_ATTRIBUTE_CLASS`, `LDR_ATTRIBUTE_SHARED`, `LDR_ATTRIBUTE_DEFAULT`.

> [!warning] CLASS and SHARED attributes are not supported in the server-side loader
> `server_class_installer::install_class` calls `er_set(ER_LDR_CLASS_NOT_SUPPORTED)` or `ER_LDR_SHARED_NOT_SUPPORTED` if `cmd_spec->attr_type` is CLASS or SHARED. Only instance attributes (`LDR_ATTRIBUTE_ANY` / `LDR_ATTRIBUTE_DEFAULT`) are accepted.

## Data row (instance_line)

Each data row is a space-separated list of `constant` values. At end of line:
```cpp
m_driver.get_object_loader().finish_line();
m_driver.get_semantic_helper().reset_after_line();
```

## Constant types (union value semantic type)

The grammar's `%union` has:

```c
%union {
  int int_val;
  string_type *string;
  constant_type *constant;
  object_ref_type *obj_ref;
  constructor_spec_type *ctor_spec;
  class_command_spec_type *cmd_spec;
};
```

`constant_type` holds a `LDR_TYPE` tag and value. Supported `%type <constant>` non-terminals:

| Non-terminal | Tokens / pattern | LDR_TYPE |
|---|---|---|
| `ansi_string` | `Quote SQS_String_Body Quote` | `LDR_STR` |
| `dq_string` | `DQuote DQS_String_Body DQuote` | `LDR_STR` |
| `nchar_string` | `NQuote SQS_String_Body Quote` | `LDR_NSTR` |
| `bit_string` | `BQuote … Quote` / `XQuote … Quote` | `LDR_BSTR` / `LDR_XSTR` |
| `sql2_date` | `DATE_ Quote … Quote` | `LDR_DATE` |
| `sql2_time` | `TIME Quote … Quote` | `LDR_TIME` |
| `sql2_timestamp` | `TIMESTAMP Quote … Quote` | `LDR_TIMESTAMP` |
| `sql2_timestampltz` | `TIMESTAMPLTZ Quote … Quote` | `LDR_TIMESTAMPLTZ` |
| `sql2_timestamptz` | `TIMESTAMPTZ Quote … Quote` | `LDR_TIMESTAMPTZ` |
| `sql2_datetime` | `DATETIME Quote … Quote` | `LDR_DATETIME` |
| `sql2_datetimeltz` | `DATETIMELTZ Quote … Quote` | `LDR_DATETIMELTZ` |
| `sql2_datetimetz` | `DATETIMETZ Quote … Quote` | `LDR_DATETIMETZ` |
| `utime` | `UTIME Quote … Quote` | `LDR_TIMESTAMP` (alias) |
| `monetary` | currency_symbol `REAL_LIT`\|`INT_LIT` | `LDR_MONETARY` |
| `object_reference` | `OBJECT_REFERENCE` OID | `LDR_OID` |
| `set_constant` | `SET_START_BRACE constant_list SET_END_BRACE` | `LDR_COLLECTION` |
| `system_object_reference` | `REF_USER`\|`REF_CLASS` string | `LDR_SYS_USER`/`LDR_SYS_CLASS` |

Currency tokens: `DOLLAR_SYMBOL`, `YEN_SYMBOL`, `WON_SYMBOL`, `EURO_SYMBOL`, and 15+ others — one token per supported currency for monetary literals.

## Time literal tokens

The lexer has specialised time literal tokens for partial time formats:
- `TIME_LIT1` through `TIME_LIT4` — `HH:MM:SS`, `HH:MM`, etc.
- `TIME_LIT31`, `TIME_LIT42` — format variants with fractional seconds

These are matched directly by the flex rules rather than requiring date keyword prefix.

## Relationship to `csql_grammar.y`

| Feature | `csql_grammar.y` | `load_grammar.yy` |
|---------|----------------|-----------------|
| Bison skeleton | Legacy C | Modern C++ (`lalr1.cc`) |
| Parse tree | `PT_NODE` tagged union | No parse tree — direct callbacks |
| Actions | Build PT_NODE, pass up rule stack | Call `driver.get_object_loader()` / `driver.get_class_installer()` directly |
| Size | ~646 KB | Much smaller |
| Error recovery | `PT_ERROR`, accumulate in PARSER_CONTEXT | `error_handler::on_failure` / `on_error` |
| Side of wire | Client-only | SA or server (CS) |

The key architectural difference: `load_grammar.yy` does **not** build a parse tree. Grammar actions call the driver's `class_installer` and `object_loader` interfaces directly — a streaming / event-driven model.

## Modifying the grammar

- Edit `load_grammar.yy` and/or `load_lexer.l`.
- Regenerate with bison 3.x: `bison -d load_grammar.yy` and `flex load_lexer.l` (or let CMake do it via its bison/flex rules).
- Set `CUBRID_LOADER_DEBUG=1` to enable bison's built-in trace for debugging shift/reduce behaviour.

## Related

- [[components/loaddb|loaddb]] — hub page
- [[components/loaddb-driver|loaddb-driver]] — how the driver drives this grammar
- [[components/loaddb-executor|loaddb-executor]] — what the grammar actions call
- [[components/parser|parser]] — the SQL grammar (different toolchain approach)
