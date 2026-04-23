---
type: component
parent_module: "[[modules/src|src]]"
path: "src/sp/"
status: developing
purpose: "How a SQL CALL statement or method reference in a query is dispatched to the cubpl::executor from the XASL query executor"
key_files:
  - "pl_executor.cpp/hpp — cubpl::executor (fetch_args_peek + execute)"
  - "pl_signature.cpp/hpp — pl_signature (type discriminator: PL_TYPE_JAVA_SP, PL_TYPE_PLCSQL, PL_TYPE_INSTANCE_METHOD, PL_TYPE_CLASS_METHOD)"
  - "pl_session.hpp — get_session(), create_and_push_stack()"
  - "sp_constants.hpp — PL_TYPE_IS_METHOD macro; METHOD_MAX_RECURSION_DEPTH = 15"
  - "method_invoke_group.cpp/hpp — shared with src/method/, groups multiple method calls in one XASL pass"
tags:
  - component
  - cubrid
  - sp
  - dispatch
related:
  - "[[components/sp|sp]]"
  - "[[components/sp-jni-bridge|sp-jni-bridge]]"
  - "[[components/query-executor|query-executor]]"
  - "[[components/xasl|xasl]]"
  - "[[components/method]]"
  - "[[Query Processing Pipeline]]"
created: 2026-04-23
updated: 2026-04-23
---

# SP Method Dispatch — CALL to cubpl::executor

## Dispatch Path

```
SQL: CALL proc(a, b)
  → parser: PT_METHOD_CALL node
  → XASL generation: METHOD_CALL_NODE inside XASL_NODE
  → query_executor.c: qexec_execute_mainblock()
      ↓
  evaluates METHOD_CALL_NODE regu-variable
      ↓
  method/src: resolves pl_signature for this proc name
      ↓
  cubpl::executor ctor (pl_signature &sig)
      ├── get_session() — finds or creates PL_SESSION for this thread
      └── session::create_and_push_stack(thread_p) — allocates execution_stack
      ↓
  executor::fetch_args_peek(regu_variable_list, val_desc, obj_oid, tuple)
      — iterates REGU_VARIABLE_LIST, calls fetch_peek_dbval(), validates types
      ↓
  executor::execute(DB_VALUE &return_value)
      (see sp-jni-bridge for detail)
      ↓
  return_value written back into the XASL regu-variable result slot
```

## PL_TYPE Discriminator

`pl_signature::type` selects the execution path:

| `PL_TYPE` value | Meaning | Execution |
|-----------------|---------|-----------|
| `PL_TYPE_JAVA_SP` | Java stored procedure | `cubpl::executor` → `SP_CODE_INVOKE` |
| `PL_TYPE_PLCSQL` | PL/CSQL stored procedure | Same path; `transaction_control` forced true |
| `PL_TYPE_INSTANCE_METHOD` | Object instance method | `PL_TYPE_IS_METHOD` macro; shared with `src/method/` |
| `PL_TYPE_CLASS_METHOD` | Class (static) method | Same |

`PL_TYPE_IS_METHOD(type)` covers both instance and class methods; these go through `method_invoke_group` rather than a direct `executor::execute()`.

## Two fetch_args_peek Overloads

`executor` exposes two argument-fetching entry points:

1. `fetch_args_peek(regu_variable_list_node *, VAL_DESCR *, OID *, QFILE_TUPLE)` — used from the query executor when the call site is inside a query (e.g., `SELECT my_func(col) FROM t`). Walks the `REGU_VARIABLE_LIST` and peeks each value via `fetch_peek_dbval()`.

2. `fetch_args_peek(std::vector<std::reference_wrapper<DB_VALUE>> args)` — used for direct `CALL proc(...)` statements where values are already materialized.

Both validate every `DB_VALUE` against `is_supported_dbtype()` before adding to `m_args`.

## Recursion Limit

`SP_CONSTANTS.hpp` defines `METHOD_MAX_RECURSION_DEPTH = 15`. Each recursive SP call allocates a new `execution_stack` on a new `cubthread::entry` worker. The session tracks all stacks in `exec_stack_map_type`; `top_stack()` returns the innermost frame.

> [!warning] Recursive SPs consume worker threads
> Each recursive invocation borrows one thread from the server thread pool for the duration of the call. Deep recursion under high concurrency risks thread starvation.

## OUT / INOUT Argument Write-back

After `executor::execute()` returns, the caller retrieves `m_out_args` via `get_out_args()`. These are `DB_VALUE` copies unpacked from the Java response. The query executor writes them back to the `REGU_VARIABLE` host-variable slots so the calling SQL can read the modified values.

## CALL from Trigger or SP Body

When a SP itself executes SQL (via `SP_CODE_INTERNAL_JDBC` callbacks), the Java side sends `callback_prepare` / `callback_execute` requests. The C side services these inside `response_callback_command()` using a fresh thread-entry context obtained from the current `execution_stack`. These SQL calls can themselves trigger further SP calls — each adds a new stack frame, bound by `METHOD_MAX_RECURSION_DEPTH`.

## Related

- [[components/sp|sp]] — hub
- [[components/sp-jni-bridge|sp-jni-bridge]] — wire-level detail
- [[components/query-executor|query-executor]] — calls fetch_args_peek / execute
- [[components/xasl|xasl]] — METHOD_CALL_NODE definition
- [[components/method]] — instance/class method path (forward-ref)
- [[Query Processing Pipeline]]
