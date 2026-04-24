---
type: source
title: "CUBRID timezones/ — IANA tzdata + Build Toolchain"
source_type: directory
source_path: "~/dev/cubrid/timezones/"
ingested: 2026-04-23
status: summarized
tags:
  - source
  - cubrid
  - timezone
  - i18n
related:
  - "[[modules/timezones]]"
  - "[[components/timezone-build]]"
  - "[[modules/locales]]"
created: 2026-04-23
updated: 2026-04-23
---

# Source: `timezones/`

CUBRID's bundled IANA timezone data and the toolchain that compiles it into the engine-loadable `libcubrid_timezones` shared library.

## Read scope
- `timezones/CMakeLists.txt`
- `timezones/make_tz.sh` (driver script — Linux)
- `timezones/tzlib/` (C compiler that turns IANA tzdata into a CUBRID shared lib)
- `timezones/tzdata/` (upstream IANA tzdata snapshot — africa, antarctica, asia, australasia, backward, etcetera, europe, northamerica, iso3166.tab, leapseconds, …)
- Windows `loclib_win_*` skeletons noted

## Pages produced
- [[modules/timezones|timezones (module page)]]
- [[components/timezone-build|timezone-build]]

## Pipeline shape
```
tzdata (IANA region files)
        │
        │  tzlib/ (in-tree C compiler)
        │  + make_tz.sh
        ▼
   generated C source → gcc/cl.exe → libcubrid_timezones.{so,dll}
        │
        ▼
   loaded by [[components/server-boot|server-boot]] for TIMESTAMPTZ / DATETIMETZ types
```

## Cross-cuts
- Engine TZ-aware types resolved here: `PT_TYPE_TIMESTAMPTZ`, `PT_TYPE_DATETIMETZ`, `PT_TYPE_TIMESTAMPLTZ` (see [[components/parse-tree]]).
- TZ data load order is fixed at server boot — replacing the .so live is not safe (engine caches offsets per-session).

## Follow-ups
- Verify exact IANA tzdata version (likely embedded in `tzdata/version` or a comment header) — would let us flag drift from upstream.
- The CDC API ([[components/cubrid-log-cdc]]) emits TZ-aware columns — sender + reader must use the same tzlib build, worth a callout.

## Source location
`~/dev/cubrid/timezones/` (read directly from the source tree).
