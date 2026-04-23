---
type: component
parent_module: "[[modules/contrib|contrib]]"
path: "contrib/collectd-plugin/, contrib/gdb_debugging_scripts/, contrib/pystatdump/, contrib/coverage/, contrib/cubmemc/"
status: developing
purpose: "Contributor-maintained monitoring, profiling, and debug tooling for CUBRID — separate from the engine build"
tags:
  - component
  - cubrid
  - observability
  - debugging
  - monitoring
  - contrib
related:
  - "[[modules/contrib|contrib]]"
  - "[[components/monitor|monitor]]"
  - "[[components/perfmon|perfmon]]"
  - "[[sources/cubrid-contrib|cubrid-contrib]]"
created: 2026-04-23
updated: 2026-04-23
---

# Contrib Observability & Debug Tools

Five `contrib/` subdirectories serve monitoring, statistical analysis, code coverage, and live debugging. None is compiled into the main CUBRID binary; each is a stand-alone tool or script.

## collectd-plugin — `contrib/collectd-plugin/`

A [collectd](https://collectd.org/) external plugin that monitors CUBRID broker transaction throughput.

- Requires: collectd 4.3.1+; CUBRID 2008 R1.3+ source (needs `cubridcs` library and broker source)
- Output artifact: `cubrid_broker.so` — loaded as a collectd plugin
- Build: edit `COLLECTD_SRC_DIR` and `CUBRID_LIBRARY` in the `Makefile`, then `make`
- Surfaces per-broker transaction counters into collectd's RRD/graphite pipeline

Adjacency: [[components/broker-impl|broker-impl]] (the process being measured), [[components/monitor|monitor]] (the engine-internal stats subsystem it complements).

> [!note] The README for collectd-plugin is written in Korean (EUC-KR encoding). The build is manually configured via `Makefile` variables — no autotools integration.

## gdb_debugging_scripts — `contrib/gdb_debugging_scripts/`

GDB helper scripts (`.gdb` / Python GDB scripts) for live or post-mortem debugging of `cub_server`, `cub_master`, and CAS processes. Typical contents include pretty-printers for CUBRID-internal structures (`PT_NODE`, `XASL_NODE`, `PAGE_BUFFER`, lock tables).

No README was found at inventory time. The cloud/ `Dockerfile` references these by installing `gdb` in the debug container and auto-loading scripts from the CUBRID install's `src/` directory.

## pystatdump — `contrib/pystatdump/`

Python 3.6+ graphical statistics visualizer for CUBRID's `cubrid statdump` output, combined with Linux `sar` and `iostat` data.

| Capability | Detail |
|-----------|--------|
| Input types | CUBRID statdump files, SAR output, iostat output |
| Derived series | `_ACCUM` (integral) and `_DELTA` (per-interval) computed automatically |
| Multi-file mixing | Samples from different sources merged into one data container |
| Filter file | Restrict which series are loaded |
| Operation modes | `stack` (overlay same stat across versions), `sim1` (similarity vs. reference), `sim2` (divergence between two runs) |
| Dependencies | Python 3.6, `scipy`, `matplotlib`, `tkinter` |
| Headless use | `export MPLBACKEND="agg"` before running |

Entry point: `python3.6 pystatdump.py <args>`

Adjacency: [[components/stats-collection|stats-collection]] (the statdump data source), [[components/perfmon|perfmon]] (underlying stats primitives).

## coverage — `contrib/coverage/`

Tooling to generate and view source-level code coverage reports for CUBRID.

Workflow:

1. Build CUBRID with `--enable-coverage` (`configure` flag)
2. Run test scenarios — `.gcda`/`.gcno` files accumulate
3. Run `contrib/coverage/mkcoverage [src/broker | src | …]` from the repo root
4. Produces `coverage.xcov` archive
5. Download to Windows; open with `CODE_COVERAGE/view_coverage.bat` (IE-based viewer) or configure another browser

> [!note] This predates lcov-html toolchains popular today. The Windows-based viewer (`view_coverage.bat`) is IE-dependent — modern workflow would pipe `.gcda` through `lcov --capture` → `genhtml`.

## cubmemc — `contrib/cubmemc/`

A memcached-protocol gateway for CUBRID. Allows memcached clients to read/write CUBRID data using the binary or text memcached protocol, potentially enabling caching layers or legacy memcached clients to use CUBRID as a backend.

No README was found at inventory time. The directory exists in the repo tree; further investigation needed to determine the current implementation state.

## Summary Table

| Tool | Language | Requires | Status note |
|------|----------|----------|-------------|
| collectd-plugin | C | collectd 4.3.1+, cubridcs lib | Korean README; manual Makefile |
| gdb_debugging_scripts | GDB / Python | GDB 7.0+ | No README found |
| pystatdump | Python 3.6 | scipy, matplotlib | Active; three analysis modes |
| coverage | Bash + Windows bat | CUBRID coverage build | IE-era viewer; still functional |
| cubmemc | C (likely) | CUBRID + memcached clients | No README found; status unclear |

## Related

- [[components/monitor|monitor]] — engine-internal stats; collectd-plugin and pystatdump consume its output
- [[components/perfmon|perfmon]] — underlying perf counter primitives
- [[components/stats-collection|stats-collection]] — `cubrid statdump` data that pystatdump ingests
- [[components/broker-impl|broker-impl]] — the broker process collectd-plugin monitors
- [[modules/contrib|contrib]] — parent module page
