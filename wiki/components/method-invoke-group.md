---
type: component
parent_module: "[[modules/src|src]]"
path: "src/sp/method_invoke_group.hpp — src/sp/method_invoke_group.cpp"
status: active
purpose: "Batches and dispatches one scan step's worth of method/SP calls; shared by src/method/ (scan path) and src/sp/ (CALL path)"
key_files:
  - "src/sp/method_invoke_group.hpp — class declaration"
  - "src/sp/method_invoke_group.cpp — begin/prepare/execute/reset/end implementation"
public_api:
  - "method_invoke_group(pl_signature_array *sig) — constructor; one per S_METHOD_SCAN node"
  - "begin() — claims pl_connection from pool"
  - "prepare(args) — packs + sends METHOD_REQUEST_ARG_PREPARE to CAS"
  - "execute(args) — sends METHOD_REQUEST_INVOKE; runs response loop; stores results"
  - "reset(bool is_end_query) — clears result vector for reuse"
  - "end() — releases pl_connection back to pool"
  - "get_return_value(int index) → DB_VALUE & — result for method #index"
  - "get_num_methods() → int"
  - "get_error_msg() / set_error_msg()"
tags:
  - component
  - cubrid
  - method
  - sp
  - dispatch
related:
  - "[[components/method|method]]"
  - "[[components/sp|sp]]"
  - "[[components/sp-method-dispatch|sp-method-dispatch]]"
  - "[[components/method-scan|method-scan]]"
  - "[[components/sp-jni-bridge|sp-jni-bridge]]"
created: 2026-04-23
updated: 2026-04-23
---

# `method_invoke_group` — Shared Method Dispatch Struct

`cubmethod::method_invoke_group` is the central dispatch object for invoking one or more methods (C builtin or Java) that correspond to a single row in an `S_METHOD_SCAN`. It is **physically located in `src/sp/`** but used by both `src/method/` (scan path) and `src/sp/` (top-level `CALL` path).

## Ownership Model

One `method_invoke_group` is created per `cubscan::method::scanner` instance (i.e., per `S_METHOD_SCAN` node in the XASL). It is allocated with `new` in `scanner::init()` and deleted in `scanner::clear(is_final=true)`.

```cpp
// scanner::init()
m_method_group = new cubmethod::method_invoke_group(sig_array);
m_method_group->begin();   // claim pl_connection
```

## Lifecycle

```
begin()         → claim cubpl::connection from pool
prepare(args)   → pack METHOD_REQUEST_ARG_PREPARE; send to CAS/SA
execute(args)   → pack METHOD_REQUEST_INVOKE; send; wait for result
  [per row]       → response loop: handle METHOD_REQUEST_CALLBACK
                 → store return values in m_result_vector
reset(false)    → clear m_result_vector for next row (reuse)
...             [repeated per scan row]
reset(true)     → full clear including execution_stack state
end()           → release pl_connection to pool
```

## Internal State

| Field | Type | Role |
|---|---|---|
| `m_id` | `METHOD_GROUP_ID` (uint64_t) | Unique group identifier — used as key in CAS-side `runtime_args` map |
| `m_stack` | `cubpl::execution_stack*` | Borrowed from `pl_session`; tracks depth and interrupt state |
| `m_sig_array` | `cubpl::pl_signature_array*` | Array of `pl_signature` objects (one per method column) |
| `m_result_vector` | `std::vector<DB_VALUE>` | Return values after `execute()`; read by `get_return_value(i)` |
| `m_is_running` | `bool` | Guards against double-end |
| `m_err_msg` | `std::string` | Error message from response loop |

## Relationship to `pl_signature`

Each entry in `m_sig_array` is a `cubpl::pl_signature` that carries:
- `type` — `PL_TYPE_JAVA_SP` / `PL_TYPE_PLCSQL` / `PL_TYPE_INSTANCE_METHOD`
- `name` — canonical method name
- `ext.method.arg_pos[]` — positional remapping of DB_VALUE arguments

For `PL_TYPE_INSTANCE_METHOD`, arg[0] is always the target object OID; subsequent args are method parameters.

## Shared Use: scan path vs. CALL path

| Context | Creator | Lifetime |
|---|---|---|
| `S_METHOD_SCAN` (query) | `cubscan::method::scanner::init()` | Per-scan-open → scan-close |
| `CALL` statement | `cubpl::executor` (indirectly) | Per statement execution |

Both paths ultimately call `execute()`, which sends data through the `cubpl::connection` and processes the response loop — so error handling, auth changes, and interrupt detection behave identically.

## Related

- [[components/method|method]] — scanner creates and owns this object
- [[components/method-scan|method-scan]] — lifecycle wrapper
- [[components/sp|sp]] — owns `method_invoke_group.cpp`; connection pool used by `begin()`/`end()`
- [[components/sp-jni-bridge|sp-jni-bridge]] — argument packing details
- [[components/sp-method-dispatch|sp-method-dispatch]] — XASL → executor path for `CALL` statements
