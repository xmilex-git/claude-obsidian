---
type: source
title: "CUBRID src/communication/ — Network Interface Layer"
path: "src/communication/"
ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - communication
  - network
  - rpc
related:
  - "[[components/communication|communication]]"
  - "[[components/packer|packer]]"
  - "[[components/request-response|request-response]]"
  - "[[components/connection|connection]]"
---

# Source: `src/communication/`

Ingested 2026-04-23. Directory owns the **client/server network interface layer** sitting directly above raw CSS TCP (`src/connection/`): request dispatch table, server-side handler registration, client-side request sending, method/xs callback glue, packer/unpacker integration, and per-request histogram instrumentation.

## What Was Read

| File | Notes |
|------|-------|
| `AGENTS.md` | Complete — authoritative directory description |
| `network.h` | Full — `NET_SERVER_REQUEST_LIST` X-macro (~250+ request constants) |
| `network_request_def.hpp` | Full — `net_request` struct, `net_req_act` flags, `net_server_func` typedef |
| `network_common.cpp` | Full — request-name string table, `get_net_request_name()` |
| `network_sr.c` (excerpt) | 700 lines — `net_server_init()` dispatch table population |
| `network_cl.c` (excerpt) | 100 lines — CS-mode-only guard, deferred query IDs, packer import |
| `network_interface_cl.c` (excerpt) | 180 lines — CS/SA dual-mode client interface, enter/exit server |
| `network_callback_cl.hpp` | Full — `xs_pack_and_queue`, `xs_send_queue` templates |
| `network_callback_sr.hpp` | Full — `pack_data`, `xs_callback_send/receive`, bidirectional send templates |
| `network_histogram.hpp` | Full — `net_histo_ctx`, `net_histogram_entry`, CS-only guard |
| `src/base/packer.hpp` | Full — `cubpacking::packer`/`unpacker`, `packing_packer`/`packing_unpacker` aliases |

## Key Findings

- `NET_SERVER_REQUEST_LIST` is an X-macro that expands into both the enum (via `NET_SERVER_REQUEST_ITEM(name) name,`) and the string table (via `#name`). ~250 request constants covering boot, transaction, locator, heap, btree, query, session, ES, replication, HA, JSP, parameters, vacuum, and more.
- `net_Requests[]` is a `static` array of `net_request` structs indexed by request code, populated in `net_server_init()`. Each entry holds `action_attribute` (bitmask) and `processing_function` (callback pointer).
- The `net_req_act` bitmask flags — `CHECK_DB_MODIFICATION`, `CHECK_AUTHORIZATION`, `SET_DIAGNOSTICS_INFO`, `IN_TRANSACTION`, `OUT_TRANSACTION` — are combined per request to control pre/post-processing behavior in `server_support.c` before the handler runs.
- `packing_packer` / `packing_unpacker` (aliases for `cubpacking::packer` / `cubpacking::unpacker`) live in `src/base/packer.hpp` and are used throughout the network layer for both callback data and XASL streams.
- Callback templates (`xs_pack_and_queue`, `xs_callback_send_args`) are header-only variadic templates that call `packer.set_buffer_and_pack_all()` — the same variadic packing idiom used by XASL stream.
- `network_histogram.hpp` is CS-mode-only. It wraps a `std::array<net_histogram_entry, NET_SERVER_REQUEST_END>` that records count, bytes sent, bytes received, and elapsed time per request code.
- No page-server / log-server server-to-server RPC was found in this directory. HA-related requests (`NET_SERVER_REPL_*`, `NET_SERVER_LOGWR_GET_LOG_PAGES`) are ordinary client-facing request slots handled by `srepl_*` and `slogwr_*` functions.

## Pages Created

- [[components/communication]] — component hub
- [[components/packer]] — packer/unpacker abstraction
- [[components/request-response]] — dispatch table and handler model
