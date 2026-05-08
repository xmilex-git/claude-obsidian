---
type: component
parent_module: "[[components/compat|compat]]"
path: "src/compat/dbtype_def.h, src/compat/db_macro.c, src/compat/dbtype_function.h"
status: active
purpose: "Universal tagged-union value container — DB_VALUE — used on both client and server sides for every SQL scalar, including NULL, all numeric types, strings, dates, sets, JSON, and LOBs"
key_files:
  - "dbtype_def.h (DB_VALUE struct + DB_TYPE enum + DB_DATA union + all sub-type structs)"
  - "db_macro.c (db_make_*() constructors + db_get_*() accessors + coercion helpers)"
  - "dbtype_function.h (DB_MAKE_*/DB_GET_* macro aliases + db_value_* extern declarations)"
  - "db_value_printer.cpp (DB_VALUE → string for debug/output)"
tags:
  - component
  - cubrid
  - db-value
  - compat
related:
  - "[[components/compat|compat]]"
  - "[[components/client-api|client-api]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/parser|parser]]"
  - "[[components/xasl-generation|xasl-generation]]"
created: 2026-04-23
updated: 2026-05-08
---

# `DB_VALUE` — Universal Value Container

`DB_VALUE` is the single structure used everywhere in CUBRID to pass SQL typed values between functions, across the client–server boundary, into and out of query evaluation, and through the storage layer.

It is defined in `src/compat/dbtype_def.h` and is part of the public ABI: applications link against it directly.

## Structure layout

```c
/* src/compat/dbtype_def.h */

typedef struct db_value DB_VALUE;
struct db_value
{
  DB_DOMAIN_INFO domain;    /* type tag + null flag + precision/scale/collation */
  DB_DATA        data;      /* the actual value — union of all SQL types */
  need_clear_type need_clear; /* bool: must db_value_clear() free data? */
};
```

### `DB_DOMAIN_INFO` — discriminant

`DB_DOMAIN_INFO` is itself a union of three views of the same 8-byte prefix:

| View | Fields | Used for |
|------|--------|----------|
| `general_info` | `is_null` (u8), `type` (u8) | All types — `is_null` tested first |
| `numeric_info` | `is_null`, `type`, `precision` (u8), `scale` (u8) | `DB_TYPE_NUMERIC` |
| `char_info` | `is_null`, `type`, `length` (int), `collation_id` (int) | String and bit types |

`type` maps to `DB_TYPE` enum values (see below). `is_null = 1` means SQL NULL regardless of `data`.

### `DB_DATA` — value payload

`DB_DATA` is a union of all possible SQL value representations:

```c
union db_data
{
  int             i;          /* DB_TYPE_INTEGER */
  short           sh;         /* DB_TYPE_SHORT / DB_TYPE_SMALLINT */
  DB_BIGINT       bigint;     /* DB_TYPE_BIGINT (int64_t) */
  float           f;          /* DB_TYPE_FLOAT */
  double          d;          /* DB_TYPE_DOUBLE */
  void           *p;          /* DB_TYPE_POINTER (method args only) */
  DB_OBJECT      *op;         /* DB_TYPE_OBJECT (MOP handle) */
  DB_TIME         time;       /* DB_TYPE_TIME (unsigned int, seconds since midnight) */
  DB_DATE         date;       /* DB_TYPE_DATE (unsigned int, Julian day) */
  DB_TIMESTAMP    utime;      /* DB_TYPE_TIMESTAMP (Unix timestamp) */
  DB_TIMESTAMPTZ  timestamptz;/* DB_TYPE_TIMESTAMPTZ */
  DB_DATETIME     datetime;   /* DB_TYPE_DATETIME (date + ms) */
  DB_DATETIMETZ   datetimetz; /* DB_TYPE_DATETIMETZ */
  DB_MONETARY     money;      /* DB_TYPE_MONETARY (double + DB_CURRENCY) */
  DB_COLLECTION  *set;        /* DB_TYPE_SET / MULTISET / SEQUENCE */
  DB_COLLECTION  *collect;    /* alias */
  DB_MIDXKEY     midxkey;     /* DB_TYPE_MIDXKEY — multi-column index key */
  DB_ELO         elo;         /* DB_TYPE_BLOB / DB_TYPE_CLOB (external LOB) */
  int            error;       /* DB_TYPE_ERROR (method return) */
  DB_IDENTIFIER  oid;         /* DB_TYPE_OID (pageid/slotid/volid) */
  DB_NUMERIC     num;         /* DB_TYPE_NUMERIC (16-byte BCD-like buf) */
  DB_CHAR        ch;          /* DB_TYPE_STRING / CHAR / BIT / VARBIT / ENUM */
  DB_RESULTSET   rset;        /* DB_TYPE_RESULTSET (uint64 handle) */
  DB_ENUM_ELEMENT enumeration;/* DB_TYPE_ENUMERATION */
  DB_JSON        json;        /* DB_TYPE_JSON (JSON_DOC* + schema_raw) */
};
```

### `DB_CHAR` — string representation (three styles)

Strings are the most complex member: `DB_CHAR` is itself a union with three representations selected by `info.style`:

| Style | Constant | When used | Key fields |
|-------|----------|-----------|-----------|
| `SMALL_STRING` | 0 | Very short strings (fit in struct) | `sm.size` + `sm.buf[DB_SMALL_CHAR_BUF_SIZE]` — no heap alloc |
| `MEDIUM_STRING` | 1 | Most strings | `medium.buf` (const char *), `medium.size`, `medium.compressed_buf` |
| `LARGE_STRING` | 2 | Very large strings | `large.str` (DB_LARGE_STRING *) |

`medium.compressed_buf` holds a zlib-compressed copy when compression is active; `compressed_size = DB_NOT_YET_COMPRESSED (0)` means not yet attempted, `DB_UNCOMPRESSABLE (-1)` means compression made it larger.

## `DB_TYPE` enum — 41 SQL types

Types are assigned stable numeric values (ABI contract):

| Value | Constant | SQL type |
|-------|----------|---------|
| 0 | `DB_TYPE_NULL` / `DB_TYPE_UNKNOWN` | SQL NULL |
| 1 | `DB_TYPE_INTEGER` | INTEGER |
| 2 | `DB_TYPE_FLOAT` | FLOAT |
| 3 | `DB_TYPE_DOUBLE` | DOUBLE |
| 4 | `DB_TYPE_STRING` / `DB_TYPE_VARCHAR` | VARCHAR |
| 5 | `DB_TYPE_OBJECT` | object reference (MOP) |
| 6–8 | `DB_TYPE_SET / MULTISET / SEQUENCE` | collection types |
| 10 | `DB_TYPE_TIME` | TIME |
| 11 | `DB_TYPE_TIMESTAMP` / `DB_TYPE_UTIME` | TIMESTAMP |
| 12 | `DB_TYPE_DATE` | DATE |
| 13 | `DB_TYPE_MONETARY` | MONETARY |
| 18 | `DB_TYPE_SHORT` / `DB_TYPE_SMALLINT` | SMALLINT |
| 22 | `DB_TYPE_NUMERIC` | NUMERIC(p,s) |
| 23 | `DB_TYPE_BIT` | BIT(n) |
| 24 | `DB_TYPE_VARBIT` | BIT VARYING |
| 25 | `DB_TYPE_CHAR` | CHAR(n) |
| 26–27 | `DB_TYPE_NCHAR_DEPRECATED` / `DB_TYPE_VARNCHAR_DEPRECATED` | Deprecated — slots preserved for ABI only |
| 31 | `DB_TYPE_BIGINT` | BIGINT |
| 32 | `DB_TYPE_DATETIME` | DATETIME |
| 33 | `DB_TYPE_BLOB` | BLOB (ELO) |
| 34 | `DB_TYPE_CLOB` | CLOB (ELO) |
| 35 | `DB_TYPE_ENUMERATION` | ENUM |
| 36–39 | `DB_TYPE_TIMESTAMPTZ` / `TIMESTAMPLTZ` / `DATETIMETZ` / `DATETIMELTZ` | Timezone-aware variants |
| 40 | `DB_TYPE_JSON` | JSON |

> [!warning] DB_TYPE values are on-disk and in serialized XASL
> These integer values appear in heap page headers, B-tree key buffers, and XASL byte streams. Never renumber them; never remove. Adding new types must append after `DB_TYPE_JSON = 40` and update `DB_TYPE_LAST`.

Internal-use-only types (not part of public SQL):
- `DB_TYPE_VARIABLE (14)`, `DB_TYPE_SUB (15)` — domain internal
- `DB_TYPE_POINTER (16)`, `DB_TYPE_ERROR (17)` — method argument passing
- `DB_TYPE_VOBJ (19)`, `DB_TYPE_OID (20)` — virtual object / raw OID
- `DB_TYPE_DB_VALUE (21)` — ESQL host variable
- `DB_TYPE_RESULTSET (28)`, `DB_TYPE_MIDXKEY (29)`, `DB_TYPE_TABLE (30)` — query internal

## Construction: `db_make_*`

All constructors are in `db_macro.c`. Pattern:

```c
/* Scalar types — direct assignment, need_clear = false */
int db_make_int (DB_VALUE *value, const int num);
int db_make_short (DB_VALUE *value, const short num);
int db_make_bigint (DB_VALUE *value, const DB_BIGINT num);
int db_make_float (DB_VALUE *value, const float num);
int db_make_double (DB_VALUE *value, const double num);
int db_make_null (DB_VALUE *value);

/* Date/time — packed into unsigned int fields */
int db_make_time (DB_VALUE *value, const int hour, const int minute, const int second);
int db_make_date (DB_VALUE *value, const int month, const int day, const int year);
int db_make_timestamp (DB_VALUE *value, const DB_TIMESTAMP *timeval);
int db_make_datetime (DB_VALUE *value, const DB_DATETIME *datetime);
int db_make_datetimetz (DB_VALUE *value, const DB_DATETIMETZ *datetimetz);

/* Strings — store pointer, need_clear = false (caller owns buf) */
int db_make_string (DB_VALUE *value, const char *str);
int db_make_varchar (DB_VALUE *value, const int max_char_length, const char *str,
                     const int char_str_byte_size, const int codeset, const int collation);
int db_make_char (DB_VALUE *value, const int char_length, const char *str,
                  const int char_str_byte_size, const int codeset, const int collation);

/* String copy — allocates, need_clear = true */
int db_make_string_copy (DB_VALUE *value, const char *str);

/* Numeric */
int db_make_numeric (DB_VALUE *value, const DB_C_NUMERIC num, const int precision, const int scale);

/* Collections — pointer stored, need_clear = true (set is ref-counted) */
int db_make_set (DB_VALUE *value, DB_C_SET *set);
int db_make_multiset (DB_VALUE *value, DB_C_SET *set);
int db_make_sequence (DB_VALUE *value, DB_C_SET *set);

/* JSON — pointer stored */
int db_make_json (DB_VALUE *value, JSON_DOC *json_document, bool need_clear);

/* OID */
int db_make_oid (DB_VALUE *value, const OID *oid);
```

> [!warning] Copy vs. borrow
> Most `db_make_*` store the **pointer** directly (`need_clear = false`). The caller retains ownership and must not free the buffer while the `DB_VALUE` is alive. `db_make_string_copy` is the exception: it allocates and sets `need_clear = true`. JSON also accepts a `need_clear` flag.

## Access: `db_get_*`

```c
int             db_get_int (const DB_VALUE *value);
DB_C_SHORT      db_get_short (const DB_VALUE *value);
DB_BIGINT       db_get_bigint (const DB_VALUE *value);
DB_CONST_C_CHAR db_get_string (const DB_VALUE *value);
DB_C_FLOAT      db_get_float (const DB_VALUE *value);
DB_C_DOUBLE     db_get_double (const DB_VALUE *value);
DB_OBJECT      *db_get_object (const DB_VALUE *value);
DB_COLLECTION  *db_get_set (const DB_VALUE *value);
DB_DATE        *db_get_date (const DB_VALUE *value);
DB_TIME        *db_get_time (const DB_VALUE *value);
DB_TIMESTAMP   *db_get_timestamp (const DB_VALUE *value);
DB_DATETIME    *db_get_datetime (const DB_VALUE *value);
DB_MONETARY    *db_get_monetary (const DB_VALUE *value);
DB_C_NUMERIC    db_get_numeric (const DB_VALUE *value);
OID            *db_get_oid (const DB_VALUE *value);
JSON_DOC       *db_get_json_document (const DB_VALUE *value);
DB_CONST_C_CHAR db_get_char (const DB_VALUE *value);   /* CHAR-typed accessor (since PR #7102, 1-arg) */
```

Macro aliases: `DB_GET_INT(v)`, `DB_GET_STRING(v)`, etc. (`dbtype_function.h`).

> [!key-insight] `db_get_char` is a thin pointer accessor
> `db_get_char` returns the underlying `data.ch.sm.buf` (small) or `data.ch.medium.buf` (medium) directly. It does **not** count characters. Pre-PR-#7102 the function had an `int *length` out-parameter populated by an `intl_char_count` UTF-8 scan; 9 of 11 call sites discarded the value. Post-PR-#7102 (`05a7befd8`, 2026-05-08) the signature is single-arg and the wasted scan is gone — callers that genuinely need a character count must call `intl_char_count` themselves. Byte size remains available via `db_get_string_size`. The `DB_GET_STRING_PRECISION` macro was also removed as dead.

## Inspection

```c
DB_TYPE db_value_domain_type (const DB_VALUE *value); /* the DB_TYPE tag */
bool    db_value_is_null (const DB_VALUE *value);      /* is_null flag */
bool    db_value_need_clear (const DB_VALUE *value);   /* need_clear flag */
int     db_value_precision (const DB_VALUE *value);
int     db_value_scale (const DB_VALUE *value);

/* Macro shortcuts */
DB_VALUE_DOMAIN_TYPE(value)   /* → db_value_domain_type */
DB_IS_NULL(value)             /* → db_value_is_null */
```

## Lifecycle

```c
/* Initialize in place — sets type + precision/scale, clears null flag */
int db_value_domain_init (DB_VALUE *value, DB_TYPE type,
                           const int precision, const int scale);

/* Copy a DB_VALUE — deep copy for heap-allocated data */
int db_value_clone (DB_VALUE *src, DB_VALUE *dest);
DB_VALUE *db_value_copy (DB_VALUE *value);   /* heap-allocated copy */

/* Free internal data if need_clear; reset is_null = 1 */
int db_value_clear (DB_VALUE *value);

/* Clear then free the DB_VALUE struct itself */
int db_value_free (DB_VALUE *value);
```

> [!warning] Always call `db_value_clear` on non-trivial values
> Any `DB_VALUE` holding a string copy, a set, or a JSON document has `need_clear = true`. Failing to call `db_value_clear` leaks that memory. For scalar types (int, double, date), `db_value_clear` is a no-op but still safe to call.

`pr_clear_value()` is an alias for `db_value_clear()` used in server-side code (`object_primitive.c`). Both call the same implementation.

## Comparison and coercion

```c
int db_value_equal (const DB_VALUE *v1, const DB_VALUE *v2);   /* 1 = equal */
int db_value_compare (const DB_VALUE *v1, const DB_VALUE *v2); /* DB_EQ, DB_LT, DB_GT, … */
int db_value_coerce (const DB_VALUE *src, DB_VALUE *dest,
                     const DB_DOMAIN *desired_domain);
```

`DB_VALUE_COMPARE_RESULT` enum: `DB_LT=-1`, `DB_EQ=0`, `DB_GT=1`, `DB_NE=2`, `DB_UNK=-2`, `DB_SUBSET=-3`, `DB_SUPERSET=3`.

## Where DB_VALUE flows in the engine

```
PT_VALUE node (parser)     ──► info.value.db_value  (DB_VALUE embedded in PT_NODE)
       │
       ▼
REGU_VARIABLE (xasl)       ──► type == TYPE_DBVAL   (DB_VALUE literal in XASL)
       │
       ▼
qexec / fetch_val_list     ──► evaluates → DB_VALUE result
       │
       ▼
db_query_get_tuple_value   ──► DB_VALUE returned to client
```

The same `DB_VALUE` type is used in all layers. On the server, `db_private_alloc` provides thread-local arena backing for large sub-fields; on the client, heap or stack.

## Related

- Parent: [[components/compat|compat]]
- [[components/regu-variable|regu-variable]] — `REGU_VARIABLE` wraps / produces `DB_VALUE` in XASL plans
- [[components/query-executor|query-executor]] — evaluates `DB_VALUE` in `fetch_val_list`, aggregates
- [[components/parser|parser]] — `PT_VALUE` nodes embed a `DB_VALUE` literal
- [[components/xasl-generation|xasl-generation]] — emits `REGU_VARIABLE` with `DB_VALUE` literals
- [[components/client-api|client-api]] — client-facing functions that return/accept `DB_VALUE`
- Source: [[sources/cubrid-src-compat|cubrid-src-compat]]
