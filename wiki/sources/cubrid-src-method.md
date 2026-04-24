---
created: 2026-04-23
type: source
title: "CUBRID src/method/ — Method/SP Invocation from Queries"
source_path: "src/method/"
date_ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - method
  - scan
related:
  - "[[components/method|method]]"
  - "[[components/method-invoke-group|method-invoke-group]]"
  - "[[components/method-scan|method-scan]]"
  - "[[components/sp|sp]]"
  - "[[components/scan-manager|scan-manager]]"
---

# Source: `src/method/`

Ingested 2026-04-23. Source: CUBRID engine, `/Users/song/DEV/cubrid/src/method/`.

## Files Read

- `AGENTS.md` — canonical architecture overview
- `method_scan.hpp / .cpp` — scanner class, S_METHOD_SCAN backend
- `method_scan.hpp` — METHOD_SCAN_ID alias, union-safety notes
- `query_method.cpp` — method_dispatch (CS/SA), builtin invoke, vobj fixup
- `method_struct_invoke.hpp` — header + prepare_args packable objects
- `method_invoke_group.hpp` (in src/sp/) — cubmethod::method_invoke_group
- `method_callback.hpp` — callback_handler, server-side JDBC callbacks
- `method_query_handler.hpp` — query_handler, prepare/execute/result

## Pages Created

- [[components/method]] — hub
- [[components/method-invoke-group]] — shared dispatch struct
- [[components/method-scan]] — S_METHOD_SCAN backend

## Key Findings

1. **Dual heritage**: `src/method/` serves both CUBRID's legacy C-language object methods (`CREATE METHOD`) and modern Java SPs/PL-CSQL when they appear in a query scan context.
2. **Shared `method_invoke_group`**: The dispatch struct lives in `src/sp/` but is used by `src/method/`'s scanner. There is no duplication — one struct handles both paths.
3. **Scan-time invocation**: Arguments are pre-computed by the query engine into a `qfile_list_id`; the scanner reads one tuple per row and fires the invoke group.
4. **Build-mode split**: Server side owns the scanner and invoke group; client/SA side owns `method_dispatch`, `callback_handler`, and `query_handler`.
5. **vobj fixup**: OID/VOBJ arguments must be fixed up to DB_TYPE_OBJECT before `obj_send_array()` — ported from `cursor.c`, noted as a legacy concern.
