---
created: 2026-04-23
type: source
title: "CUBRID contrib/ tree"
path: "contrib/"
date_ingested: 2026-04-23
status: complete
tags:
  - source
  - cubrid
  - contrib
  - drivers
  - deployment
related:
  - "[[modules/contrib|contrib]]"
  - "[[components/contrib-language-drivers|contrib-language-drivers]]"
  - "[[components/contrib-observability|contrib-observability]]"
  - "[[components/contrib-deployment|contrib-deployment]]"
---

# Source: CUBRID contrib/ Tree

Ingested: 2026-04-23. Source directory: `contrib/` at CUBRID repo root.

## What Was Read

| Path | Type | Notes |
|------|------|-------|
| `cloud/README.md` | Full read | Kubernetes StatefulSet deployment guide |
| `cloud/Dockerfile` | Full read | Multi-stage CentOS builder |
| `cloud/build.sh` | Partial read | Docker/kubectl wrapper script |
| `cloud/cubrid-statefulset.yaml` | Partial read | K8s StatefulSet + Services |
| `python/README` | Full read | CUBRIDdb + Django backend overview |
| `python-obsolete/README` | Full read | Legacy Python driver (Korean-encoded portions) |
| `php5/README` | Full read | PHP extension overview |
| `perl/README` | Full read | DBD::cubrid DBI driver |
| `pystatdump/README.md` | Full read | Statistics visualizer |
| `coverage/README` | Full read | Code coverage report tool |
| `collectd-plugin/README` | Full read | Korean; collectd plugin for broker stats |
| `init.d/cubrid` | Full read | SysV init script source |
| `ruby/` | Directory listed | No README found |
| `adodotnet/` | Directory listed | No README found |
| `hibernate/` | Directory listed | No README found |
| `gdb_debugging_scripts/` | Directory listed | No README found |
| `cubmemc/` | Directory listed | No README found |
| `bash/` | Directory listed | No README found |
| `getshardid/` | Directory listed | No README found |
| `msg/` | Directory listed | No README found |

## Inventory (19 directories)

```
adodotnet/          — ADO.NET driver (.NET clients)
bash/               — bash tab-completion for cubrid CLI
cloud/              — Docker + Kubernetes StatefulSet
collectd-plugin/    — collectd plugin (broker transaction stats)
coverage/           — lcov/xcov code coverage tooling
cubmemc/            — memcached gateway for CUBRID
gdb_debugging_scripts/ — GDB helpers for engine debugging
getshardid/         — shard key hash calculator
hibernate/          — Hibernate ORM dialect
init.d/             — SysV init script
msg/                — contributed locale/message files
perl/               — DBD::cubrid DBI driver
php4/               — legacy PHP4 driver (archived)
php5/               — PHP5 PECL extension
pystatdump/         — Python3 statdump/SAR/iostat visualizer
python/             — CUBRIDdb Python DB API 2.0 + Django
python-obsolete/    — older Python 2 driver (archived)
readme/             — contributor orientation notes
ruby/               — Ruby adapter
```

## Pages Created from This Source

- [[modules/contrib]] — module hub page (replaces placeholder)
- [[components/contrib-language-drivers]] — Python, PHP, Perl, Ruby, ADO.NET, Hibernate
- [[components/contrib-observability]] — collectd-plugin, pystatdump, coverage, gdb_debugging_scripts, cubmemc
- [[components/contrib-deployment]] — cloud/ Docker/K8s, init.d, bash completion

## Key Facts Captured

- All language drivers wrap CCI or JDBC — no driver speaks the CSS wire protocol directly.
- `cloud/` is the only contrib component referenced by CI; uses multi-stage CentOS Docker build.
- `init.d/cubrid` skips operations when `ha_mode=yes/on/role-change` to avoid interfering with HA coordination.
- `pystatdump` computes both `_ACCUM` (integral) and `_DELTA` (per-interval) derived series and supports three analysis modes.
- `python-obsolete/` and `python/` coexist; the current driver is `python/`.

## Follow-ups

- Confirm contents of `ruby/`, `adodotnet/`, `hibernate/`, `gdb_debugging_scripts/`, `cubmemc/`, `getshardid/` — no README found at ingest time.
- `cubmemc/` status is unclear (may be prototype/abandoned).
- `coverage/` uses IE-based viewer — document lcov-based modern alternative.
