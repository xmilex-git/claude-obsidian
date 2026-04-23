---
type: component
parent_module: "[[modules/src|src]]"
path: "src/base/packer.hpp / src/base/packer.cpp"
status: developing
purpose: "Type-safe binary serialization: cubpacking::packer writes primitive and composite values into a byte buffer; cubpacking::unpacker reads them back. Used by network interface layer, XASL stream, and method callbacks."
key_files:
  - "src/base/packer.hpp (packer + unpacker class declarations; variadic templates)"
  - "src/base/packer.cpp (implementations)"
public_api:
  - "cubpacking::packer — writes into a fixed char* buffer"
  - "cubpacking::unpacker — reads from a const char* buffer"
  - "packing_packer / packing_unpacker — C-file-friendly aliases (using declarations)"
  - "packer::set_buffer_and_pack_all(ExtBlk&, Args...) — size+extend+pack in one call"
  - "packer::pack_int / pack_bool / pack_short / pack_bigint / pack_db_value / pack_string / pack_c_string / pack_oid / pack_buffer_with_length"
  - "unpacker::unpack_int / unpack_bool / unpack_short / unpack_bigint / unpack_db_value / unpack_string / unpack_c_string / unpack_oid / unpack_buffer_with_length"
  - "packer::get_all_packed_size(Args...) — compute needed size before allocating"
  - "packer::pack_all(Args...) / unpacker::unpack_all(Args...) — variadic bulk ops"
  - "packable_object — virtual interface for types that self-describe their pack/unpack"
tags:
  - component
  - cubrid
  - packer
  - serialization
  - network
related:
  - "[[components/communication|communication]]"
  - "[[components/xasl-stream|xasl-stream]]"
  - "[[components/request-response|request-response]]"
  - "[[Memory Management Conventions]]"
  - "[[sources/cubrid-src-communication|cubrid-src-communication]]"
created: 2026-04-23
updated: 2026-04-23
---

# `cubpacking::packer` / `unpacker` — Binary Serialization

Defined in `src/base/packer.hpp`. The packer/unpacker pair is CUBRID's primary mechanism for converting typed C++ values to and from a flat byte stream. It is used by the network interface layer (argument marshalling for all `NET_SERVER_*` requests), by the XASL serialization path, and by method/SP callback data exchange.

## Core Classes

### `cubpacking::packer`

Writes into a caller-provided `char*` buffer (or auto-extended `cubmem::extensible_block`). Not thread-safe; one packer per serialization context.

```cpp
namespace cubpacking {
  class packer {
   public:
    packer (char *storage, const size_t amount);
    void set_buffer (char *storage, const size_t amount);

    // Primitive pack operations
    void pack_int (const int value);
    void pack_bool (const bool value);
    void pack_short (const short value);
    void pack_bigint (const std::int64_t &value);
    void pack_bigint (const std::uint64_t &value);
    void pack_db_value (const db_value &value);
    void pack_string (const std::string &str);
    void pack_c_string (const char *str, const size_t str_size);
    void pack_oid (const OID &oid);
    void pack_buffer_with_length (const char *stream, const size_t length);

    // Corresponding size queries (offset-aware alignment)
    size_t get_packed_int_size (size_t curr_offset);
    size_t get_packed_db_value_size (const db_value &value, size_t curr_offset);
    // ... etc.

    // Bulk variadic API
    template <typename... Args>
    size_t get_all_packed_size (Args&&... args);

    template <typename... Args>
    void pack_all (Args&&... args);

    // Compute size, extend extensible_block, then pack — the most common pattern:
    template <typename ExtBlk, typename... Args>
    void set_buffer_and_pack_all (ExtBlk &eb, Args&&... args);

    // Append to existing extensible_block content:
    template <typename ExtBlk, typename... Args>
    void append_to_buffer_and_pack_all (ExtBlk &eb, Args&&... args);

    bool has_error () const;
    size_t get_current_size ();
    const char *get_curr_ptr ();
    void align (const size_t req_alignment);
    void delegate_to_or_buf (const size_t size, or_buf &buf);
  };
}
```

### `cubpacking::unpacker`

Reads from a caller-provided `const char*` buffer. Mirrors packer exactly — operations must be called in the same order as the corresponding pack calls.

```cpp
namespace cubpacking {
  class unpacker {
   public:
    unpacker (const char *storage, const size_t amount);
    unpacker (const cubmem::block &blk);

    void unpack_int (int &value);
    void unpack_bool (bool &value);
    void unpack_short (short &value);
    void unpack_bigint (std::int64_t &value);
    void unpack_db_value (db_value &value);
    void unpack_string (std::string &str);
    void unpack_c_string (char *str, const size_t max_str_size);
    void unpack_oid (OID &oid);
    void unpack_buffer_with_length (char *stream, const size_t max_length);

    template <typename... Args>
    void unpack_all (Args&&... args);

    bool has_error () const;
    bool is_ended ();
    void delegate_to_or_buf (const size_t size, or_buf &buf);
  };
}
```

### `packable_object` — Self-Describing Types

Types that know how to pack and unpack themselves implement the `cubpacking::packable_object` virtual interface. This lets `packer::pack_overloaded(const packable_object &)` dispatch to the object's own logic. Used for complex structures (e.g., query plan nodes) that need to be serialized without external knowledge of their layout.

## C-File Aliases

```cpp
using packing_packer   = cubpacking::packer;
using packing_unpacker = cubpacking::unpacker;
```

These aliases exist because `indent` (the C formatter used in CI) is confused by namespaces in `.c` files. All `.c` files use `packing_packer` / `packing_unpacker` directly.

## Variadic Bulk Pattern

The most common network-layer usage:

```cpp
packing_packer packer;
cubmem::extensible_block eb;
// 1. compute total size, 2. extend eb, 3. pack all args in one call:
packer.set_buffer_and_pack_all (eb, arg1, arg2, arg3, ...);
```

Supported argument types for variadic dispatch (resolved via `pack_overloaded` / `get_packed_size_overloaded` overloads):

| Type | Notes |
|------|-------|
| `int`, `bool`, `short` | Primitives |
| `std::int64_t`, `std::uint64_t` | 64-bit integers |
| `db_value` | Universal CUBRID value (all DB types) |
| `std::string`, `const char*` | Strings (length-prefixed, small vs. large branches) |
| `OID` | Object identifier |
| `cubmem::block` | Raw byte block |
| `std::vector<T>` | Any packable T; prefixed by count (bigint) |
| `std::reference_wrapper<T>` | Transparent wrapper — unpacks via `.get()` |
| `packable_object` | Virtual dispatch via `pack/get_packed_size` |

## Alignment

Packer methods call `align()` internally where needed. The `get_packed_*_size(curr_offset)` functions account for required alignment so callers can compute exact sizes before allocating.

## Relationship to `or_buf`

`packer` is intended to gradually replace the older `OR_BUF` (object representation buffer) C structure. `delegate_to_or_buf()` exists as an escape hatch for code that still depends on `or_buf` semantics (e.g., some object representation paths).

## Where Packer Is Used

| Consumer | Usage |
|----------|-------|
| `network_callback_cl.hpp` | `xs_pack_and_queue` — pack method callback data client-side |
| `network_callback_sr.hpp` | `pack_data`, `xs_callback_send_args` — pack method callback data server-side |
| `network_interface_cl.c` / `network_interface_sr.c` | Pack/unpack arguments for all `NET_SERVER_*` requests |
| `src/xasl/xasl_stream.cpp` | XASL plan serialization (same packer infrastructure) |
| Various `src/sp/` files | SP argument marshalling |

## Integration

- [[components/communication|communication]] — packer is the serialization engine for all request arguments
- [[components/xasl-stream|xasl-stream]] — uses the same packer infrastructure for XASL node trees; similar offset-based approach
- [[Memory Management Conventions]] — packer writes into `db_private_alloc`-managed memory or `cubmem::extensible_block`; no RAII ownership transfers occur inside packer itself

## Related

- Parent: [[modules/src|src]]
- Source: [[sources/cubrid-src-communication|cubrid-src-communication]]
