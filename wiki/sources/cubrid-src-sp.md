---
type: source
title: "CUBRID src/sp/ — Stored Procedure JNI Bridge"
source_path: "src/sp/"
ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - sp
  - stored-procedure
  - jni
related:
  - "[[components/sp|sp]]"
  - "[[components/sp-jni-bridge|sp-jni-bridge]]"
  - "[[components/sp-method-dispatch|sp-method-dispatch]]"
  - "[[components/sp-protocol|sp-protocol]]"
  - "[[modules/pl_engine|pl_engine]]"
---

# Source: `src/sp/` — Stored Procedure JNI Bridge

**Ingested:** 2026-04-23
**Files read:** `AGENTS.md`, `pl_executor.hpp/cpp`, `pl_session.hpp`, `pl_connection.hpp`, `pl_signature.hpp`, `pl_comm.c/h`, `pl_sr.cpp/h`, `pl_sr_jvm.h`, `pl_execution_stack_context.hpp`, `sp_catalog.hpp`, `sp_constants.hpp`

## Summary

`src/sp/` bridges the C++ CUBRID engine (`cub_server`) and the Java PL engine (`cub_pl`). It is responsible for SP catalog DDL, JVM process lifecycle, connection pooling, argument marshalling, and the invoke/callback/result protocol.

Key finding: despite the "JNI" label, there is no in-process JNI. `cub_pl` is a separate OS process containing the JVM. All inter-language communication is done over Unix domain sockets (Linux/macOS) or TCP (Windows/remote).

## Pages Created

- [[components/sp|sp]] — hub page: full architecture, lifecycle, catalog, error propagation
- [[components/sp-jni-bridge|sp-jni-bridge]] — invocation mechanics, DB_VALUE marshalling, interrupt handling
- [[components/sp-method-dispatch|sp-method-dispatch]] — XASL → executor dispatch, recursion, OUT arg write-back
- [[components/sp-protocol|sp-protocol]] — transport selection, SP_CODE opcodes, METHOD_CALLBACK loop

## Key Insights

1. **JVM is an OS process, not embedded**: `server_manager` forks `cub_pl` via `create_child_process()`. Communication is sockets only.
2. **Bidirectional callback loop**: Java SP bodies can call back into C (SQL prepare/execute/fetch/transaction) via `SP_CODE_INTERNAL_JDBC`, holding the connection open for the full SP duration.
3. **Connection pool epoch**: After `cub_pl` restart, the epoch counter transparently triggers reconnection without caller changes.
4. **Transaction control flag**: PL/CSQL always gets `transaction_control=true`; Java SPs use `PRM_ID_PL_TRANSACTION_CONTROL` parameter.
5. **Type gaps**: `BLOB`, `CLOB`, `JSON`, timezone-aware timestamps, and `ENUMERATION` are unsupported at the bridge layer.
