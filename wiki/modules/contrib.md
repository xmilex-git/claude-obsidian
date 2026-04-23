---
type: module
path: "contrib/"
status: developing
purpose: "Contributor-maintained language drivers, observability plugins, and operational tooling; NOT part of the engine build by default"
key_subdirs:
  - "python/ — Python DB API 2.0 driver (CUBRIDdb + Django backend)"
  - "python-obsolete/ — older Python driver (BSD; archived)"
  - "php5/ — PHP extension for CUBRID (PECL-style)"
  - "php4/ — legacy PHP4 driver (archived)"
  - "perl/ — DBD::cubrid DBI driver"
  - "ruby/ — Ruby adapter"
  - "adodotnet/ — ADO.NET driver for .NET clients"
  - "hibernate/ — Hibernate ORM dialect"
  - "cloud/ — Docker + Kubernetes StatefulSet deployment"
  - "init.d/ — SysV init script for Linux service management"
  - "bash/ — bash tab-completion for cubrid CLI"
  - "collectd-plugin/ — collectd plugin reporting broker transaction stats"
  - "gdb_debugging_scripts/ — GDB helper scripts for engine debugging"
  - "pystatdump/ — Python3 graphical visualizer for statdump/SAR/iostat output"
  - "coverage/ — lcov/xcov code-coverage report tooling"
  - "cubmemc/ — memcached gateway for CUBRID"
  - "getshardid/ — utility to compute shard key hash for shard-broker routing"
  - "msg/ — contributed locale/message files"
  - "readme/ — top-level contrib overview directory"
tags:
  - module
  - cubrid
  - contrib
  - drivers
  - tooling
related:
  - "[[components/contrib-language-drivers|contrib-language-drivers]]"
  - "[[components/contrib-observability|contrib-observability]]"
  - "[[components/contrib-deployment|contrib-deployment]]"
  - "[[modules/cubrid-cci|cubrid-cci]]"
  - "[[modules/cubrid-jdbc|cubrid-jdbc]]"
  - "[[sources/cubrid-contrib|cubrid-contrib]]"
created: 2026-04-23
updated: 2026-04-23
---

# contrib/ — Contributor Tools and Drivers

`contrib/` holds everything that lives outside the engine proper: language-binding drivers, observability scripts, cloud deployment configs, and Linux packaging helpers. Nothing in this directory is compiled into the main CUBRID build by default — each item is independently built, packaged, or used at operator discretion.

## Groupings

| Group | Directories | Wiki page |
|-------|-------------|-----------|
| Language drivers | python, python-obsolete, php5, php4, perl, ruby, adodotnet, hibernate | [[components/contrib-language-drivers]] |
| Observability / debugging | collectd-plugin, gdb_debugging_scripts, pystatdump, coverage, cubmemc | [[components/contrib-observability]] |
| Deployment / operations | cloud, init.d, bash | [[components/contrib-deployment]] |
| Misc utilities | getshardid, msg, readme | (inline below) |

## Misc Utilities

- **getshardid** — standalone C program that computes the shard key hash for a given value, useful for manually routing queries against a [[components/shard-broker|shard-broker]] deployment.
- **msg** — contributed locale/message additions or overrides separate from the main `msg/` tree.
- **readme** — top-level directory holding general contributor orientation notes.

## Key Points

- All language drivers wrap either [[modules/cubrid-cci|cubrid-cci]] (C-level CCI client) or [[modules/cubrid-jdbc|cubrid-jdbc]] (JDBC). No driver speaks the CSS wire protocol directly.
- The cloud/ tooling is the only part of contrib/ that is actively referenced by CI (Docker image builds).
- `python-obsolete/` predates `python/` by roughly two years (authors differ); the current `python/` driver adds Django backend support and Python 3 compatibility.
- Licenses vary: engine is GPL v2+, but API drivers (PHP, Python) are BSD.

## Related

- [[components/contrib-language-drivers]] — per-driver details
- [[components/contrib-observability]] — monitoring and debug tools
- [[components/contrib-deployment]] — Docker / init.d / bash
- [[modules/cubrid-cci|cubrid-cci]] — C client library that most drivers depend on
- [[modules/cubrid-jdbc|cubrid-jdbc]] — JDBC driver used by Hibernate
