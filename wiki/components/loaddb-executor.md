---
type: component
parent_module: "[[modules/src|src]]"
path: "src/loaddb/"
status: active
purpose: "Server-side execution: class registration, per-row DB_VALUE conversion, heap record construction, and bulk insert via locator_multi_insert_force"
key_files:
  - "load_server_loader.cpp (server_class_installer, server_object_loader)"
  - "load_db_value_converter.cpp (string → DB_VALUE dispatch table)"
  - "load_class_registry.cpp (class_entry, attribute, class_registry)"
  - "load_error_handler.cpp (per-line error tracking)"
tags:
  - component
  - cubrid
  - loaddb
  - heap
  - executor
related:
  - "[[components/loaddb|loaddb]]"
  - "[[components/loaddb-grammar|loaddb-grammar]]"
  - "[[components/loaddb-driver|loaddb-driver]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/btree|btree]]"
created: 2026-04-23
updated: 2026-05-08
---

# loaddb Executor (server-side loader)

The execution layer for CS-mode loaddb. Grammar actions in [[components/loaddb-grammar|loaddb-grammar]] call two abstract interfaces — `class_installer` and `object_loader` — which are implemented server-side by `server_class_installer` and `server_object_loader` in `load_server_loader.cpp`.

Hub: [[components/loaddb|loaddb]].

## Class installer (`server_class_installer`)

Triggered by `%class` and `%id` directives in the input file.

### `install_class(name, cmd_spec)`

Full registration path:

```
install_class(class_name, cmd_spec)
  └── register_class_with_attributes(class_name, cmd_spec)
        ├── locate_class(lower_case_name, class_oid)     -- xlocator_find_class_oid w/ BU_LOCK
        ├── heap_attrinfo_start(class_oid, -1, ...)       -- load all attribute representations
        ├── heap_scancache_quick_start_root_hfid(...)
        ├── heap_get_class_record(class_oid, PEEK)        -- read class record
        ├── get_class_attributes(attrinfo, attr_type, ...)-- select instance/class/shared attrs
        ├── sort by or_attribute.def_order               -- maintain schema definition order
        ├── build attribute → or_attribute map
        ├── check missing NOT NULL columns               -- ER_OBJ_ATTRIBUTE_CANT_BE_NULL if gap
        └── class_registry.register_class(name, clsid, class_oid, attributes)
```

Key: attributes are sorted by `def_order` (schema definition order), not by position in the `%class` column list. When a column list is provided in the input file, it is matched by name against `attr_map`, building the correct index mapping.

### Backwards-compatibility class lookup

```
locate_class(class_name, class_oid)
  ├── if name has dot → xlocator_find_class_oid directly
  ├── if system class → xlocator_find_class_oid directly
  └── else (no schema qualifier, pre-11.2 unload file)
        ├── prepend session user_name → try xlocator_find_class_oid
        └── if CLIENT_TYPE_ADMIN_LOADDB_COMPAT_UNDER_11_2
              └── locate_class_for_all_users()
                    -- heap scan of _db_user, try user.table for each user
                    -- returns LC_CLASSNAME_DELETED if ambiguous (>1 match)
```

## DB_VALUE converter (`load_db_value_converter.cpp`)

A type-dispatch table mapping `(LDR_TYPE, DB_TYPE)` → conversion function. Each function has the signature:

```cpp
int to_db_X(const char *str, size_t str_size, const attribute *attr, db_value *val);
```

Representative conversions:

| LDR type | DB_TYPE | Function | Notes |
|----------|---------|----------|-------|
| `LDR_INT` | `DB_TYPE_SHORT` | `to_db_short` | Checks `MAX_DIGITS_FOR_SHORT = 5` |
| `LDR_INT` | `DB_TYPE_INTEGER` | `to_db_int` | Checks `MAX_DIGITS_FOR_INT = 10` |
| `LDR_INT` | `DB_TYPE_BIGINT` | `to_db_bigint` | Checks `MAX_DIGITS_FOR_BIGINT = 19` |
| `LDR_STR` | `DB_TYPE_CHAR` | `to_db_char` | Calls `to_db_generic_char` |
| `LDR_STR` | `DB_TYPE_VARCHAR` | `to_db_varchar` | Calls `to_db_generic_char` |
| `LDR_FLOAT` | `DB_TYPE_FLOAT` | `to_db_float` | `strtod` → `db_make_float` |
| `LDR_DOUBLE` | `DB_TYPE_DOUBLE` | `to_db_double` | `strtod` → `db_make_double` |
| `LDR_NUMERIC` | `DB_TYPE_NUMERIC` | `to_db_numeric` | `numeric_coerce_string_to_num` |
| `LDR_DATE` | `DB_TYPE_DATE` | `to_db_date` | `db_date_parse_date` |
| `LDR_DATETIME` | `DB_TYPE_DATETIME` | `to_db_datetime` | `db_date_parse_datetime` |
| `LDR_TIMESTAMPTZ` | `DB_TYPE_TIMESTAMPTZ` | `to_db_timestamptz` | TZ-aware parsing |
| `LDR_STR` | `DB_TYPE_JSON` | `to_db_json` | `db_json_get_json_from_str` |
| `LDR_BSTR` | `DB_TYPE_VARBIT` | `to_db_varbit_from_bin_str` | Binary string |
| `LDR_XSTR` | `DB_TYPE_VARBIT` | `to_db_varbit_from_hex_str` | Hex string |
| `LDR_ELO_EXT` | `DB_TYPE_ELO` | `to_db_elo_ext` | External LOB |
| `LDR_ELO_INT` | `DB_TYPE_ELO` | `to_db_elo_int` | Inline LOB |
| — | any | `mismatch` | Called when LDR type / DB type are incompatible |

The conversion functions do **not** go through the SQL type-checking pipeline. Type compatibility is checked purely by LDR token kind vs. column's `DB_TYPE` at load time.

> [!key-insight] Domain precision = schema precision (since PR #7102)
> `to_db_generic_char` (load_db_value_converter.cpp) and `ldr_str_db_char` / `ldr_str_db_varchar` (load_sa_loader.cpp) initialize the loaded `DB_VALUE`'s domain via `db_value_domain_init(val, type, precision, 0)` and set `val.domain.char_info.length = precision` — using the **column's defined precision**, not the input string's character count. Pre-[[prs/PR-7102-db-get-char-intl-cleanup|PR #7102]] (`05a7befd8`, 2026-05-08) the same fields were populated with `char_count` of the input, which made every loaded row's domain disagree with the schema; consumers that read `domain.char_info.length` would see a value that varied per row instead of the constant column precision. Truncation enforcement still uses precision: byte-size short-circuit first (`byte_count <= precision` ⇒ no scan), then `intl_char_count` only if the byte size already exceeds precision.

> [!note] TODO: CBRD-21654
> A comment in `load_db_value_converter.cpp` notes that conversion functions should be reused between `load_sa_loader.cpp` and the server loader. Currently there is duplication.

## Object loader (`server_object_loader`)

Processes one data row per grammar `instance_line`.

### Per-row lifecycle

```
parser grammar action → object_loader.process_line(constant_type *cons)
  └── for each constant c in linked list:
        process_constant(c, attr[attr_index])
          ├── process_generic_constant → load_db_value_converter dispatch
          ├── process_monetary_constant
          ├── process_collection_constant  (recursive for sets)
          └── heap_attrinfo_set(class_oid, attr.id, &db_val, &m_attrinfo)

grammar action → object_loader.finish_line()
  ├── heap_attrinfo_transform_to_disk_except_lob(thread, &m_attrinfo, NULL, &new_recdes)
  ├── if no error: m_recdes_collected.push_back(move(new_recdes))
  └── clear_db_values()
```

### Batch-end flush

```
flush_records()
  ├── if HA enabled or errors filtered:
  │     for each recdes:
  │       log_sysop_start()
  │       locator_insert_force(hfid, class_oid, &recdes, ...)
  │       if error: log_sysop_abort(); on_failure()
  │       else: log_sysop_attach_to_outer(); m_rows++
  │
  └── else (normal fast path):
        log_sysop_start()
        locator_multi_insert_force(hfid, class_oid, m_recdes_collected, MULTI_ROW_INSERT, ...)
        if error: log_sysop_abort(); on_failure()
        else: log_sysop_attach_to_outer(); m_rows += recdes_collected.size()
```

`locator_multi_insert_force` inserts all accumulated records from a batch in a single call — this is the key performance path. It uses `MULTI_ROW_INSERT` op type which may generate page-level log records (not per-row), which is why the HA path falls back to per-row `locator_insert_force` for accurate replication LSAs.

### BU lock

The class OID must hold a `BU_LOCK` (Bulk-Update) before `init()` is called. This is asserted in:
```cpp
assert(lock_has_lock_on_object(&class_oid, oid_Root_class_oid, BU_LOCK));
```

BU lock prevents concurrent DDL and conflicting writes but is compatible with shared readers.

## Class registry (`load_class_registry.cpp`)

`class_registry` is a session-scoped map from `class_id` → `class_entry`.

`class_entry` holds:
- `class_name` (std::string, lowercased)
- `class_oid` (OID)
- `std::vector<const attribute*>` sorted by index (schema order)
- `is_ignored` flag (for `--ignore-class` CLI option)

`attribute` wraps an `or_attribute*` (from the class representation) plus a pre-built converter function pointer (`conv_func_t`), looked up from the `load_db_value_converter` dispatch table at registration time.

## Error handling

`error_handler` fields:
- `m_current_line_has_error` — set on any per-value conversion error; checked in `finish_line()` to discard the row
- `m_syntax_check` — mirrors `session.get_args().syntax_check`

Error severity in normal mode:
- `on_failure()` / `on_failure_with_line()` — calls `session::fail()`, all subsequent tasks abort
- `on_error()` / `on_error_with_line()` — records error but does not abort session (used in syntax-check mode or for filtered errors)
- `on_syntax_failure()` — non-aborting syntax error; discards current row

Error messages fetched from `MSGCAT_CATALOG_UTILS / MSGCAT_UTIL_SET_LOADDB` via `msgcat_message()`.

## Related

- [[components/loaddb|loaddb]] — hub and architecture overview
- [[components/loaddb-grammar|loaddb-grammar]] — the grammar that calls into this layer
- [[components/loaddb-driver|loaddb-driver]] — driver, session, worker management
- [[components/heap-file|heap-file]] — `heap_attrinfo_*`, `heap_scancache_*` implementations
- [[components/btree|btree]] — B-tree index updates triggered by `locator_insert_force`
