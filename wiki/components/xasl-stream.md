---
type: component
parent_module: "[[modules/src|src]]"
path: "src/xasl/xasl_stream.hpp + src/query/xasl_to_stream.c + src/query/stream_to_xasl.c"
status: active
purpose: "XASL serialisation protocol: client packs XASL_NODE tree to flat byte stream; server unpacks stream back to XASL_NODE tree; offset-based pointer encoding with visited-pointer dedup table"
key_files:
  - "src/xasl/xasl_stream.hpp (stx_build/stx_restore templates, alignment constants, XASL_UNPACK_INFO helpers)"
  - "src/xasl/xasl_unpack_info.hpp (XASL_UNPACK_INFO struct, visited-pointer table, extra buffers)"
  - "src/query/xasl_to_stream.c (xts_map_xasl_to_stream — client packer, !SERVER_MODE)"
  - "src/query/stream_to_xasl.c (stx_map_stream_to_xasl — server unpacker, SERVER_MODE|SA_MODE)"
  - "src/query/xasl_to_stream.h"
  - "src/query/stream_to_xasl.h"
public_api:
  - "xts_map_xasl_to_stream(xasl, stream) → int   [!SERVER_MODE]"
  - "xts_map_filter_pred_to_stream(pred, &buf, &size) → int"
  - "xts_map_func_pred_to_stream(func_pred, &buf, &size) → int"
  - "stx_map_stream_to_xasl(thread_p, &xasl_tree, use_clone, buf, size, &unpack_info) → int   [SERVER|SA]"
  - "stx_map_stream_to_filter_pred(...)"
  - "stx_map_stream_to_func_pred(...)"
  - "stx_map_stream_to_xasl_node_header(thread_p, header_p, stream)"
tags:
  - component
  - cubrid
  - xasl
  - serialization
related:
  - "[[components/xasl|xasl]]"
  - "[[components/xasl-generation|xasl-generation]]"
  - "[[components/query-executor|query-executor]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# XASL Stream — Serialisation Protocol

This component is the wire encoding that carries an `XASL_NODE` plan tree from the client process to the server process. See [[components/xasl|xasl]] for the overall picture.

## Build-mode split

```
Client (CS_MODE / SA_MODE):
  xasl_to_stream.c   #if !defined(SERVER_MODE) #error
  xts_map_xasl_to_stream()   → writes XASL_STREAM buffer

Server (SERVER_MODE / SA_MODE):
  stream_to_xasl.c   #if !defined(SERVER_MODE) && !defined(SA_MODE) #error
  stx_map_stream_to_xasl()   → reads XASL_STREAM buffer → XASL_NODE *
```

In `SA_MODE` (standalone library) both halves are linked together; the "stream" is still created and consumed in the same process to maintain code parity.

## Buffer layout

```
XASL_STREAM buffer:
┌─────────────────────────────────────────────────────────────────┐
│  XASL_STREAM_HEADER  (8 bytes, XASL_STREAM_HEADER constant)    │
│  header_size (int)                                              │
│  header data:                                                   │
│    creator_OID (OID)                                            │
│    n_oid_list  (int)                                            │
│    class_oid_list[n] (OID[])                                    │
│    repr_id_list[n]   (int[])                                    │
│    dbval_cnt   (int)                                            │
│  body_size (int)                                                │
│  body data: packed XASL_NODE tree                              │
│    ← all pointers encoded as int offsets from body start →     │
└─────────────────────────────────────────────────────────────────┘
```

Macros `GET_XASL_STREAM_HEADER_DATA`, `XASL_STREAM_BODY_PTR`, etc. navigate these sections without any struct overlay (pure pointer arithmetic).

## Alignment

All fields are written at 8-byte boundaries:

```c
const int XASL_STREAM_ALIGN_UNIT = sizeof(double);  // 8
const int XASL_STREAM_ALIGN_MASK = XASL_STREAM_ALIGN_UNIT - 1;

inline int xasl_stream_make_align(int x) {
  return (x & ~XASL_STREAM_ALIGN_MASK) + ((x & XASL_STREAM_ALIGN_MASK) ? XASL_STREAM_ALIGN_UNIT : 0);
}
```

Writes use `or_pack_*` helpers from `object_representation.h` which assert alignment (`ASSERT_ALIGN`).

## Packing (client side) — `xts_map_xasl_to_stream`

1. Compute required buffer size by a dry-run size pass.
2. Allocate a flat `char` buffer of that size (+ `STREAM_EXPANSION_UNIT` growth reserve).
3. Walk the `XASL_NODE` tree depth-first, calling `xts_save_*` per type.
4. Every pointer sub-structure is written at a new aligned offset; the field in the parent becomes `or_pack_int(offset)`.
5. Return the filled `XASL_STREAM` struct with `buffer` + `buffer_size`.

Growth: `STREAM_EXPANSION_UNIT = OFFSETS_PER_BLOCK * sizeof(int) = 4096 * 4 = 16 384 bytes`.

## Unpacking (server side) — `stx_map_stream_to_xasl`

> [!key-insight] Visited-pointer deduplication
> Shared substructures (e.g. a `REGU_VARIABLE` referenced from multiple places) must not be duplicated on unpack. `stx_mark_struct_visited(thread_p, bufptr, target)` maps the source buffer pointer to the freshly allocated server pointer. `stx_get_struct_visited_ptr` returns the cached pointer on second encounter, skipping re-allocation and re-parsing.

Steps for each `stx_restore<T>(thread_p, ptr, target)`:
1. `or_unpack_int(ptr, &offset)` — read the stored offset.
2. If `offset == 0`: `target = NULL`.
3. Else: look up `bufptr = packed_xasl + offset` in visited table.
4. If found: return cached pointer.
5. If not found: `stx_alloc_struct(thread_p, sizeof(T))` → `stx_mark_struct_visited` → `stx_build(thread_p, bufptr, *target)`.

`stx_alloc_struct` draws from `XASL_UNPACK_INFO.alloc_buf` (a single large arena allocation).

### `XASL_UNPACK_INFO` (`xasl_unpack_info.hpp`)

```c
struct xasl_unpack_info {
  char            *packed_xasl;       // start of the stream buffer
  STX_VISITED_PTR *ptr_blocks[256];   // visited-pointer hash table
  char            *alloc_buf;         // arena for unpacked structs
  int              packed_size;
  int              ptr_lwm[256];      // low-water-mark per block
  int              ptr_max[256];      // allocated capacity per block
  int              alloc_size;
  UNPACK_EXTRA_BUF *additional_buffers; // extra allocations during unpack
  int              track_allocated_bufers;
  bool             use_xasl_clone;
};
```

Visited-pointer hash: `block = ((UINTPTR)ptr / sizeof(UINTPTR)) % 256`. Linear probing within each block.

> [!warning] UNPACK_SCALE = 3
> Callers must pre-allocate the arena with at least `3 × stream_size` bytes (`UNPACK_SCALE` constant in `xasl.h`). Underestimating causes mid-unpack realloc which invalidates the `packed_xasl` pointer used throughout deserialization.

## Thread safety

`XASL_UNPACK_INFO` is stored per-thread in a `THREAD_ENTRY` slot. Accessors:
- `get_xasl_unpack_info_ptr(thread_p)` / `set_xasl_unpack_info_ptr(thread_p, ptr)`

This means deserialization is inherently single-threaded per query (the worker thread that receives the stream owns the unpack context).

## Filter predicate stream

Filtered indexes use `PRED_EXPR_WITH_CONTEXT` — a predicate without a full `XASL_NODE` wrapper. Packed via `xts_map_filter_pred_to_stream`; unpacked via `stx_map_stream_to_filter_pred`. Same offset encoding, separate stream buffer.

## Debugging / comparison

`xasl_stream.hpp` exposes comparison overloads:
```cpp
bool xasl_stream_compare(const cubxasl::json_table::column &first, ...);
bool xasl_stream_compare(const cubxasl::json_table::node &first, ...);
bool xasl_stream_compare(const cubxasl::json_table::spec_node &first, ...);
```
Used in debug builds to validate that pack→unpack is a round-trip identity.

## CS_MODE note

In `CS_MODE`, `stx_restore<T>` is a no-op (reads the offset int and sets `target = NULL`). This allows `xasl_stream.hpp` to be included in client builds for debug-comparison purposes without pulling in server allocation.

## Related

- [[components/xasl|xasl]] — hub: XASL_NODE structure and overall protocol
- [[components/xasl-generation|xasl-generation]] — builds the XASL_NODE tree that gets packed
- [[components/query-executor|query-executor]] — executes the unpacked tree
- [[Build Modes (SERVER SA CS)]]
- [[Query Processing Pipeline]]
- Source: [[sources/cubrid-src-xasl|cubrid-src-xasl]]
