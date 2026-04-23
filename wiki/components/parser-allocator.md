---
type: component
parent_module: "[[components/parser|parser]]"
path: "src/parser/parser_allocator.hpp"
status: active
purpose: "C++ block_allocator wrapper over parser_alloc; provides arena-lifetime allocation for C++ code in the parser pipeline"
key_files:
  - "parser_allocator.hpp (parser_block_allocator class)"
  - "mem_block.hpp (cubmem::block_allocator interface — src/base/)"
  - "parse_tree_cl.c (parser_alloc implementation)"
public_api:
  - "parser_block_allocator(parser_context*) — constructor"
  - "(inherits) alloc(block&, size) / dealloc(block&)"
  - "parser_alloc(parser, length) → void* — C-level arena allocator"
  - "pt_append_string(parser, old, new) → char* — append to parser-owned string"
tags:
  - component
  - cubrid
  - parser
  - memory
related:
  - "[[components/parser|parser]]"
  - "[[Memory Management Conventions]]"
  - "[[components/parse-tree|parse-tree]]"
created: 2026-04-23
updated: 2026-04-23
---

# Parser Allocator (`parser_allocator.hpp`)

Bridges the C parser arena (`parser_alloc`) to the C++ `cubmem::block_allocator` interface used by modern CUBRID C++ code in the parser pipeline.

## Lifetime model

The parser uses an arena (bump-pointer) allocator tied to a `PARSER_CONTEXT`. All memory allocated through `parser_alloc(parser, size)` is freed in a single operation when `parser_free_parser(parser)` is called. There is no per-object `free`.

```
parser_create_parser()
    │
    │  All allocs go into parser's arena
    ├─ parser_alloc(parser, N)         → void*  (never free individually)
    ├─ parser_new_node(parser, type)   → PT_NODE*
    ├─ pt_append_string(parser, s, t)  → char*
    │
    ▼
parser_free_parser(parser)             → frees entire arena at once
```

> [!key-insight] No manual free
> Code that calls `parser_alloc` must never call `free()` on the result. The arena is the only valid dealloc path. This matches CUBRID's broader [[Memory Management Conventions]] convention of using arena/pool allocators for complex object graphs.

## `parser_block_allocator` (C++ wrapper)

```cpp
class parser_block_allocator : public cubmem::block_allocator
{
public:
  parser_block_allocator() = delete;
  parser_block_allocator(parser_context *parser);

private:
  void alloc(cubmem::block &b, size_t size);   // calls parser_alloc
  void dealloc(cubmem::block &b);              // no-op (arena lifetime)

  parser_context *m_parser;
};
```

`cubmem::block_allocator` is the common interface used throughout CUBRID's C++ layer (page buffer, lock manager, etc.) for pluggable allocation. By implementing it over `parser_alloc`, C++ objects in the parser pipeline — such as JSON table nodes, analytic function lists, and XASL sub-structures allocated during generation — can use the standard C++ block-allocator API without escaping the parser's arena lifetime.

`dealloc` is a no-op: individual blocks are not freed; the arena is released as a whole.

## String management

```c
// Append new_tail to old_string in the parser arena.
// Returns a new parser-owned copy; old_string may be NULL.
char *pt_append_string(
    const PARSER_CONTEXT *parser,
    const char *old_string,
    const char *new_tail);

// Low-level variable-length byte buffer (PARSER_VARCHAR):
PARSER_VARCHAR *pt_append_bytes(parser, old_bytes, new_tail, length);
PARSER_VARCHAR *pt_append_varchar(parser, old_bytes, new_tail);
PARSER_VARCHAR *pt_append_nulstring(parser, old_bytes, new_tail);
```

`PARSER_VARCHAR` is a simple length-prefixed byte buffer (`struct parser_varchar { int length; unsigned char bytes[1]; }`), allocated in the parser arena.

## OOM recovery

When `parser_alloc` cannot satisfy an allocation, it calls `longjmp` to the `jmp_buf` saved in `parser_context.jmp_env` (set up via `PT_SET_JMP_ENV(parser)`). The handler records an out-of-memory error and returns `NULL`. All parser functions that perform allocation must set up the JMP environment at their entry point.

## Invariants

- Never call `free()` on a pointer returned by `parser_alloc`.
- `parser_block_allocator` must not outlive the `parser_context` it was constructed with.
- The `dealloc` no-op means C++ destructors of objects placed in parser-arena memory must not rely on `dealloc` being called for cleanup; use trivially-destructible types or manage destructors explicitly.

## Related

- Parent: [[components/parser|parser]]
- [[Memory Management Conventions]] — global conventions; `parser_alloc` is the parser-specific pattern
- [[components/parse-tree|parse-tree]] — all PT_NODE allocations use this arena
