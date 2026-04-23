---
type: component
parent_module: "[[modules/src|src]]"
path: "src/xasl/ (type headers) + src/query/xasl.h (main node)"
status: active
purpose: "eXecutable Algebraic Statement Language — the serializable query-plan tree that crosses the client→server wire; produced by xasl_generation, deserialized by stream_to_xasl, executed by query_executor"
key_files:
  - "src/query/xasl.h (XASL_NODE — main plan node, PROC_TYPE enum, ACCESS_SPEC_TYPE, XASL flags, XASL_STREAM struct, macros)"
  - "src/query/regu_var.hpp (regu_variable_node / REGU_VARIABLE — the expression atom)"
  - "src/xasl/xasl_predicate.hpp (PRED_EXPR, EVAL_TERM, REL_OP, BOOL_OP)"
  - "src/xasl/xasl_aggregate.hpp (AGGREGATE_TYPE / aggregate_list_node)"
  - "src/xasl/xasl_analytic.hpp (ANALYTIC_TYPE / analytic_list_node)"
  - "src/xasl/xasl_stream.hpp (stx_build/stx_restore, alignment constants, XASL_UNPACK_INFO helpers)"
  - "src/xasl/xasl_unpack_info.hpp (XASL_UNPACK_INFO — deserialisation context)"
  - "src/xasl/xasl_sp.hpp (SP_TYPE — stored-procedure invocation)"
  - "src/query/xasl_to_stream.c (client side: xts_map_xasl_to_stream)"
  - "src/query/stream_to_xasl.c (server side: stx_map_stream_to_xasl)"
  - "src/query/xasl_cache.h (XASL_CACHE_ENTRY, xcache_* API)"
  - "src/query/subquery_cache.h (SQ_CACHE, XASL_USES_SQ_CACHE)"
public_api:
  - "xts_map_xasl_to_stream(xasl, stream) → int   [client only, !SERVER_MODE]"
  - "stx_map_stream_to_xasl(thread_p, &xasl_tree, use_clone, buf, size, &unpack_info) → int   [server/SA]"
  - "stx_map_stream_to_filter_pred / stx_map_stream_to_func_pred"
  - "stx_map_stream_to_xasl_node_header"
tags:
  - component
  - cubrid
  - xasl
  - query
related:
  - "[[modules/src|src]]"
  - "[[components/parser|parser]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[components/optimizer|optimizer]]"
  - "[[components/query|query]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/xasl-stream|xasl-stream]]"
  - "[[components/regu-variable|regu-variable]]"
  - "[[components/xasl-predicate|xasl-predicate]]"
  - "[[components/xasl-aggregate|xasl-aggregate]]"
  - "[[components/xasl-analytic|xasl-analytic]]"
  - "[[Query Processing Pipeline]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# `src/xasl/` — XASL: eXecutable Algebraic Statement Language

XASL is the **wire-format query plan** that bridges client and server in CUBRID. After the [[components/parser|parser]] + [[components/xasl-generation|xasl-generation]] produce an `XASL_NODE` tree on the client, it is serialised to a flat byte stream and sent to the server, which deserialises it and hands it directly to the [[components/query-executor|query executor]]. The server never runs parser code.

> [!key-insight] Why XASL exists
> CUBRID separates parsing + optimisation (client-side, CS_MODE) from execution (server-side, SERVER_MODE). The two processes communicate over TCP. XASL is the language-neutral, pointer-free byte representation of a complete execution plan. Every pointer in the in-memory `XASL_NODE` tree becomes a stream offset; `stx_restore` resolves it back to a pointer on the server.

## Pipeline (XASL's portion)

```
parser/ + optimizer/
  ──► xasl_generation.c    PT_NODE tree → XASL_NODE tree   (client, !SERVER_MODE)
  ──► xasl_to_stream.c     XASL_NODE → flat byte stream    (client, !SERVER_MODE)
  ──► [TCP / shared memory]
  ──► stream_to_xasl.c     byte stream → XASL_NODE tree    (server, SERVER_MODE|SA_MODE)
  ──► query_executor.c     execute XASL_NODE tree           (server)
```

See [[Query Processing Pipeline]] for the full path.

## Build-mode ownership

| Guard | Owns |
|-------|------|
| `!SERVER_MODE` (client) | `xasl_to_stream.c`, `xasl_generation.c` |
| `SERVER_MODE \| SA_MODE` | `stream_to_xasl.c`, `query_executor.c` |
| **Both** (shared types) | Everything in `src/xasl/` + `src/query/xasl.h` + `src/query/regu_var.hpp` |

> [!warning] No mode-specific guards in shared headers
> `src/xasl/*.hpp` and `regu_var.hpp` must compile cleanly in all three build modes. Any `#if defined(SERVER_MODE)` block inside them covers runtime-only fields (accumulators, scan stats) and must not affect serialised structure layout. See [[Build Modes (SERVER SA CS)]].

---

## Core structures

### `XASL_NODE` (`src/query/xasl.h`)

The root of every query plan. A linked tree — each node has a `PROC_TYPE` discriminant and a corresponding proc union member.

```c
struct xasl_node {
  XASL_NODE_HEADER  header;         // xasl_flag + id (always first — sent separately)
  XASL_NODE        *next;           // sibling / sub-plan link
  PROC_TYPE         type;           // BUILDLIST_PROC, SCAN_PROC, UPDATE_PROC, …
  int               flag;           // XASL_TOP_MOST_XASL, XASL_TO_BE_CACHED, …
  QFILE_LIST_ID    *list_id;        // output list file
  OUTPTR_LIST      *outptr_list;    // output-value regu list
  ACCESS_SPEC_TYPE *spec_list;      // table/index/list access specs
  PRED_EXPR        *during_join_pred;
  PRED_EXPR        *after_join_pred;
  PRED_EXPR        *instnum_pred;   // ROWNUM / inst_num()
  REGU_VARIABLE    *limit_offset;
  REGU_VARIABLE    *limit_row_count;
  XASL_NODE        *aptr_list;      // CTEs + uncorrelated subqueries
  XASL_NODE        *dptr_list;      // correlated subqueries
  // … many more fields …
  union {
    UNION_PROC_NODE   union_;       // UNION / DIFFERENCE / INTERSECTION
    FETCH_PROC_NODE   fetch;        // OBJFETCH_PROC
    BUILDLIST_PROC_NODE buildlist;  // SELECT with GROUP BY / analytic
    BUILDVALUE_PROC_NODE buildvalue;// aggregate-only SELECT
    MERGELIST_PROC_NODE mergelist;  // sort-merge join
    HASHJOIN_PROC_NODE  hashjoin;   // hash join
    UPDATE_PROC_NODE    update;
    INSERT_PROC_NODE    insert;
    DELETE_PROC_NODE    delete_;
    CONNECTBY_PROC_NODE connect_by;
    MERGE_PROC_NODE     merge;
    CTE_PROC_NODE       cte;
  } proc;
};
```

### `PROC_TYPE` enum (complete)

Exact values from `xasl.h` (`enum` order):

| Ordinal | Constant | Usage / proc union member |
|---------|----------|--------------------------|
| 0 | `UNION_PROC` | `UNION ALL` / set union (shares `union_` proc) |
| 1 | `DIFFERENCE_PROC` | `EXCEPT` — set difference |
| 2 | `INTERSECTION_PROC` | `INTERSECT` — set intersection |
| 3 | `OBJFETCH_PROC` | Object fetch by OID (`FETCH_PROC_NODE.arg`) |
| 4 | `BUILDLIST_PROC` | SELECT result list; aggregates + analytic window functions (`BUILDLIST_PROC_NODE`) |
| 5 | `BUILDVALUE_PROC` | Aggregate-only SELECT without grouping temp file (`BUILDVALUE_PROC_NODE`) |
| 6 | `SCAN_PROC` | Leaf scan node — drives `ACCESS_SPEC_TYPE`; no dedicated proc union field |
| 7 | `MERGELIST_PROC` | Sort-merge join (`MERGELIST_PROC_NODE`) |
| 8 | `HASHJOIN_PROC` | Hash join (`HASHJOIN_PROC_NODE`) |
| 9 | `UPDATE_PROC` | Server-side UPDATE (`UPDATE_PROC_NODE`) |
| 10 | `DELETE_PROC` | Server-side DELETE (`DELETE_PROC_NODE`) |
| 11 | `INSERT_PROC` | Server-side INSERT (`INSERT_PROC_NODE`); includes ODKU |
| 12 | `CONNECTBY_PROC` | Hierarchical query (`CONNECTBY_PROC_NODE`) |
| 13 | `DO_PROC` | `DO` statement — no output |
| 14 | `MERGE_PROC` | `MERGE` statement (`MERGE_PROC_NODE`: update_xasl + insert_xasl + has_delete) |
| 15 | `BUILD_SCHEMA_PROC` | `SHOW` statement schema queries |
| 16 | `CTE_PROC` | CTE (`CTE_PROC_NODE`: non_recursive_part + recursive_part) |

`PROC_TYPE` is serialized as an integer into the XASL stream. Adding new values must maintain ordinal stability.

### `XASL_ID`

Cache identity key: SHA1 hash of the serialised stream + `time_stored` timestamp. Compared with `XASL_ID_EQ`. The `cache_flag` is not copied across the wire — it is a server-local hint.

### `XASL_NODE_HEADER`

Tiny prefix (`xasl_flag`, `id`) sent to the client as soon as a plan is cached, before re-executing. Lets the client know if multi-range optimisation is active without unpacking the full tree. Size is always `XASL_NODE_HEADER_SIZE` (two ints, currently 8 bytes).

---

## Sub-components (modular type headers)

| Page | Header | Key types |
|------|--------|-----------|
| [[components/regu-variable\|regu-variable]] | `regu_var.hpp` | `regu_variable_node`, `REGU_DATATYPE`, `ARITH_TYPE`, `FUNCTION_TYPE` |
| [[components/xasl-predicate\|xasl-predicate]] | `xasl_predicate.hpp` | `PRED_EXPR`, `EVAL_TERM`, `REL_OP`, `BOOL_OP` |
| [[components/xasl-aggregate\|xasl-aggregate]] | `xasl_aggregate.hpp` | `AGGREGATE_TYPE`, `aggregate_accumulator` |
| [[components/xasl-analytic\|xasl-analytic]] | `xasl_analytic.hpp` | `ANALYTIC_TYPE`, `ANALYTIC_EVAL_TYPE` |
| [[components/xasl-stream\|xasl-stream]] | `xasl_stream.hpp` | `stx_build`, `stx_restore`, `XASL_UNPACK_INFO` |

---

## Serialisation protocol

> [!key-insight] Offset-based pointer encoding
> Every pointer in the in-memory XASL tree is serialised as an **integer byte offset** from the start of the packed buffer. On the server, `stx_restore<T>` reads that offset, checks a visited-pointer table (to handle shared substructures), allocates a fresh `T` via `stx_alloc_struct`, marks it visited, then calls `stx_build` to populate its fields recursively. Shared sub-trees are deserialised only once; subsequent references return the cached pointer.

Key constants (from `xasl_stream.hpp`):

| Constant | Value | Meaning |
|----------|-------|---------|
| `XASL_STREAM_ALIGN_UNIT` | `sizeof(double)` = 8 | All writes/reads are 8-byte aligned |
| `OFFSETS_PER_BLOCK` | 4096 | Visited-pointer hash block size |
| `MAX_PTR_BLOCKS` | 256 | Max blocks in `XASL_UNPACK_INFO.ptr_blocks` |
| `STREAM_EXPANSION_UNIT` | `4096 * sizeof(int)` = 16 384 B | Dynamic growth step for pack buffer |
| `UNPACK_SCALE` | 3 | Assumed memory ratio: unpacked ≈ 3× packed bytes |

The buffer layout for the full XASL stream (`XASL_STREAM` struct, `xasl.h`):

```
[ XASL_STREAM_HEADER (8 bytes) ]
[ header_size (int) ][ header data: creator_OID, class OID list, repr IDs, dbval_cnt ]
[ body_size   (int) ][ body data: packed XASL_NODE tree + all sub-structures ]
```

Client-side serialiser: `xts_map_xasl_to_stream` (`xasl_to_stream.c`, `!SERVER_MODE`).
Server-side deserialiser: `stx_map_stream_to_xasl` (`stream_to_xasl.c`, `SERVER_MODE|SA_MODE`).

Both functions are compiled from source files in `src/query/` (not `src/xasl/`), but they consume the type headers from `src/xasl/`.

### Adding a field to XASL — the four-file rule

> [!warning] Serialisation/deserialisation must be kept in sync across four files
> Adding any field to an XASL structure requires touching **all four**, in order:
> 1. **`src/xasl/*.hpp` or `src/query/xasl.h`** — declare the field in the struct
> 2. **`src/query/xasl_to_stream.c`** — add `or_pack_*` / `xts_save_*` call in the client packer
> 3. **`src/query/stream_to_xasl.c`** — add `or_unpack_*` / `stx_build` call in the server unpacker
> 4. **`src/parser/xasl_generation.c`** — set the field value when constructing the plan
>
> Optionally (if the field needs server-side cache invalidation): also update `xcache_entry_get_entrysize()` in `xasl_cache.c`.
>
> A mismatch between pack and unpack causes a mis-aligned read and typically a crash or silent data corruption at deserialization time. The XASL stream is **not versioned** — client and server must be built from the same source.

### `XASL_ID` structure

```c
struct xasl_id {
  SHA1Hash   sha1;          // 20 bytes — SHA-1 of rewritten query hash text
  CACHE_TIME time_stored;   // sec + usec — plan creation timestamp
  INT32      cache_flag;    // 24-bit ref count + 8-bit status flags (server-only, NOT serialised)
};
```

`XASL_ID_EQ` compares `sha1` and `time_stored` only; `cache_flag` is excluded.
`XASL_ID_COPY` copies `sha1` and `time_stored` only.

The `time_stored` discriminates two plans with identical SQL but compiled at different times (e.g. after a schema change forced recompile). A session holds its stale `time_stored`; the mismatch in `xcache_find_xasl_id_for_execute` triggers transparent re-prepare.

---

## Access spec (`ACCESS_SPEC_TYPE`)

The "from-clause" element of a plan node. Describes which table/index/list to open and how:

```c
struct access_spec_node {
  TARGET_TYPE   type;       // TARGET_CLASS, TARGET_LIST, TARGET_JSON_TABLE, …
  ACCESS_METHOD access;     // SEQUENTIAL, INDEX, JSON_TABLE, SCHEMA, SAMPLING, …
  INDX_INFO    *indexptr;
  PRED_EXPR    *where_key;  // key filter (on index)
  PRED_EXPR    *where_pred; // residual predicate
  HYBRID_NODE   s;          // union: cls_node / list_node / set_node / …
  int           flags;      // ACCESS_SPEC_FLAG_FOR_UPDATE, NO_PARALLEL_HEAP_SCAN, …
};
```

### `TARGET_TYPE` family (complete)

| Constant | Value | Hybrid union member | Meaning |
|----------|-------|---------------------|---------|
| `TARGET_CLASS` | 1 | `cls_node` (CLS_SPEC_TYPE) | Heap/index scan of a class |
| `TARGET_CLASS_ATTR` | 2 | `cls_node` | Class attribute access |
| `TARGET_LIST` | 3 | `list_node` (LIST_SPEC_TYPE) | Scan of a temp list file (join side, subquery output) |
| `TARGET_SET` | 4 | `set_node` (SET_SPEC_TYPE) | Set expression scan |
| `TARGET_JSON_TABLE` | 5 | json_table node | JSON_TABLE() function |
| `TARGET_METHOD` | 6 | `method_node` (METHOD_SPEC_TYPE) | Method scan |
| `TARGET_REGUVAL_LIST` | 7 | `reguval_list_node` | VALUES clause (multi-row INSERT) |
| `TARGET_SHOWSTMT` | 8 | `showstmt_node` (SHOWSTMT_SPEC_TYPE) | SHOW statement |
| `TARGET_DBLINK` | 9 | `dblink_node` (DBLINK_SPEC_TYPE) | DBLink remote scan |

### `ACCESS_METHOD` family (complete)

| Constant | Meaning |
|----------|---------|
| `ACCESS_METHOD_SEQUENTIAL` | Full heap scan |
| `ACCESS_METHOD_INDEX` | B-tree index scan |
| `ACCESS_METHOD_JSON_TABLE` | JSON_TABLE scan |
| `ACCESS_METHOD_SCHEMA` | Schema (catalog) access |
| `ACCESS_METHOD_SEQUENTIAL_RECORD_INFO` | Heap scan reading page/slot metadata |
| `ACCESS_METHOD_SEQUENTIAL_PAGE_SCAN` | Page-only scan (no record data) |
| `ACCESS_METHOD_INDEX_KEY_INFO` | Index scan for key info |
| `ACCESS_METHOD_INDEX_NODE_INFO` | B-tree node info scan |
| `ACCESS_METHOD_SEQUENTIAL_SAMPLING_SCAN` | Statistical sampling scan |

---

## XASL flags (complete bitmask)

`XASL_NODE.flag` bits, checked/set/cleared via `XASL_IS_FLAGED` / `XASL_SET_FLAG` / `XASL_CLEAR_FLAG`:

| Bit | Constant | Meaning |
|-----|----------|---------|
| 0x0001 | `XASL_LINK_TO_REGU_VARIABLE` | Subquery is linked from a `REGU_VARIABLE.xasl` pointer |
| 0x0002 | `XASL_SKIP_ORDERBY_LIST` | Skip ORDER BY list (result already sorted) |
| 0x0004 | `XASL_ZERO_CORR_LEVEL` | Uncorrelated subquery (zero-correlation level) |
| 0x0008 | `XASL_TOP_MOST_XASL` | Root node of the plan tree |
| 0x0010 | `XASL_TO_BE_CACHED` | Result list file will be cached (used by XASL_USES_SQ_CACHE path) |
| 0x0020 | `XASL_HAS_NOCYCLE` | `CONNECT BY NOCYCLE` specified |
| 0x0040 | `XASL_HAS_CONNECT_BY` | Has `CONNECT BY` clause |
| 0x0080 | `XASL_MULTI_UPDATE_AGG` | Multi-table UPDATE with aggregate |
| 0x0100 | `XASL_IGNORE_CYCLES` | Ignore cycles in LEVEL-based `CONNECT BY` |
| 0x0200 | `XASL_OBJFETCH_IGNORE_CLASSOID` | Object fetch ignores class OID check |
| 0x0400 | `XASL_IS_MERGE_QUERY` | Part of a MERGE statement |
| 0x0800 | `XASL_USES_MRO` | Multi-range optimization active |
| 0x1000 | `XASL_DECACHE_CLONE` | Decache clone at end (clear accumulators etc.) |
| 0x2000 | `XASL_RETURN_GENERATED_KEYS` | Return auto-generated keys (INSERT) |
| 0x4000 | `XASL_NO_FIXED_SCAN` | Disable fixed scan optimization |
| 0x8000 | `XASL_NEED_SINGLE_TUPLE_SCAN` | EXISTS subquery — stop after one result |
| 0x10000 | `XASL_INCLUDES_TDE_CLASS` | References a TDE-encrypted class |
| 0x20000 | `XASL_SAMPLING_SCAN` | Sampling scan (TABLESAMPLE) |
| 0x40000 | `XASL_USES_SQ_CACHE` | Correlated scalar subquery result cache enabled (see [[components/subquery-cache]]) |
| 0x80000 | `XASL_NO_PARALLEL_SUBQUERY` | Parallel subquery disabled for this node |
| 0x100000 | `XASL_ANALYTIC_USES_LIMIT_OPT` | Analytic window uses LIMIT optimization |
| 0x200000 | `XASL_ANALYTIC_SKIP_SORT` | Analytic window skips sort (already sorted) |

---

## Conventions

- All structures in `src/xasl/` are in the `cubxasl` namespace (C++) with legacy C `using` aliases for the `UPPER_SNAKE` names still used throughout `src/query/`.
- Mode-specific runtime fields (`accumulator`, scan IDs, stats) are wrapped in `#if defined(SERVER_MODE) || defined(SA_MODE)`. They are **not serialised** — they are populated at execution time.
- `regu_variable_node` has a `map_regu` recursive walker; use it instead of manual traversal.
- `REGU_VARIABLE` fields that reference subquery `XASL_NODE *` are executed via the `EXECUTE_REGU_VARIABLE_XASL` macro (calls `qexec_execute_mainblock` on first access).

---

## Gotchas

- **`regu_var.hpp` is in `src/query/`, not `src/xasl/`** — confusingly, `REGU_VARIABLE` is the most complex type and lives next to the main `xasl.h`, not in the modular headers directory.
- **`UNPACK_SCALE = 3`** — allocate 3× the stream size as the working arena before unpacking; underestimating causes realloc mid-unpack.
- **Clones** — the server may clone an XASL tree for parallel execution; clone decache must clear `DB_VALUE`s tagged `clear_value_at_clone_decache` in aggregate accumulators and regu variables.
- **Filter predicate stream** — `PRED_EXPR_WITH_CONTEXT` has its own independent stream path (`xts_map_filter_pred_to_stream` / `stx_map_stream_to_filter_pred`), used by filtered indexes, independent of the main XASL stream.

---

## Related

- Parent: [[modules/src|src]]
- [[components/parser|parser]] — produces `PT_NODE` tree that xasl_generation converts to `XASL_NODE`
- [[components/xasl-generation|xasl-generation]] — `PT_NODE → XASL_NODE` translation
- [[components/optimizer|optimizer]] — selects the plan shape (access methods, join order) that xasl_generation encodes
- [[components/query|query]] — server-side query execution layer hub
- [[components/query-executor|query-executor]] — `qexec_execute_mainblock` dispatches on `PROC_TYPE`
- [[components/xasl-stream|xasl-stream]] — serialisation protocol deep dive
- [[components/xasl-cache|xasl-cache]] — server-side plan cache; stores packed stream keyed by SHA-1
- [[components/regu-variable|regu-variable]] — `REGU_VARIABLE` / expression atom
- [[components/xasl-predicate|xasl-predicate]] — `PRED_EXPR` predicate tree
- [[components/xasl-aggregate|xasl-aggregate]] — `AGGREGATE_TYPE` aggregate node
- [[components/xasl-analytic|xasl-analytic]] — `ANALYTIC_TYPE` window-function node
- [[components/subquery-cache|subquery-cache]] — correlated scalar subquery result cache (`XASL_USES_SQ_CACHE`)
- [[Query Processing Pipeline]]
- [[Build Modes (SERVER SA CS)]]
- Source: [[sources/cubrid-src-xasl|cubrid-src-xasl]]
