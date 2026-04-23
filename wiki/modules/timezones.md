---
type: module
path: "timezones/"
status: developing
purpose: "IANA timezone data + compile toolchain that produces libcubrid_timezones shared library; engine uses it for TIMESTAMPTZ / DATETIMETZ types"
key_files:
  - "timezones/tzdata/ (IANA tzdata source text files, ~2017 vintage)"
  - "timezones/CMakeLists.txt (build target: gen_timezone executable + cubrid_timezones shared lib)"
  - "timezones/make_tz.sh (post-install update script; new / extend modes)"
  - "timezones/tzlib/build_tz.sh (inner compile script: gcc -fPIC → .so)"
  - "timezones/tzlib/timezones.c (generated; not tracked in VCS)"
  - "src/base/timezone_lib_common.h (shared structs: TZ_DATA, TZ_TIMEZONE, TZ_DS_RULE, …)"
  - "src/base/tz_support.h / tz_support.c (runtime API: tz_load, tz_unload, tz_conv_tz_datetime_*)"
  - "src/base/tz_compile.h / tz_compile.c (compile-time API: timezone_compile_data)"
  - "src/executables/generate_timezone.cpp (gen_timezone binary: calls timezone_compile_data)"
tags:
  - module
  - cubrid
  - timezones
  - iana
  - datetimetz
  - timestamptz
related:
  - "[[components/server-boot|server-boot]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/timezone-build|timezone-build]]"
  - "[[sources/cubrid-timezones|cubrid-timezones]]"
created: 2026-04-23
updated: 2026-04-23
---

# `timezones/` — Timezone Data and Build Toolchain

The `timezones/` directory is a self-contained subsystem with two responsibilities:

1. **Data layer** — a vendor-bundled copy of the IANA Time Zone Database (`tzdata/`) that provides the authoritative source for all timezone rules, DST transitions, country codes, and leap-second records.
2. **Build toolchain** — a two-phase pipeline that transforms the raw text files into a C source file (`timezones.c`), then compiles that C source into the `libcubrid_timezones` shared library that the engine loads at startup.

## Why a Separate Shared Library?

Timezone data changes frequently (IANA releases several updates per year). By isolating all timezone data in a dynamically loaded library, CUBRID allows operators to update timezone data without recompiling the full engine. The engine only needs `tz_load()` / `tz_unload()` to refresh from disk.

The library name is defined in `src/base/tz_support.h`:

```c
#define LIB_TZ_NAME "libcubrid_timezones.so"   // Linux / macOS
// Windows: libcubrid_timezones.dll
// AIX:     libcubrid_timezones.a(libcubrid_timezones.so.<MAJOR>)
```

## tzdata — IANA Source Files

`timezones/tzdata/` is an embedded copy of the IANA tz database. File-level evidence (leap-seconds list through 2016, Africa data updated 2017-02-20) indicates a **~2017b / 2017c vintage**. No canonical `version` file is present in the tree.

Key files:

| File | Content |
|------|---------|
| `africa`, `europe`, `northamerica`, `asia`, `australasia`, `southamerica`, `antarctica`, `etcetera`, `pacificnew`, `backward` | Zone and Rule records (standard IANA text format) |
| `leapseconds` | 27 leap-second records through 2016-12-31 (IERS Bulletin C53) |
| `windowsZones.xml` | CLDR Windows-to-IANA zone name mapping table |
| `iso3166.tab` | ISO 3166 country codes |
| `zone.tab` | Canonical zone list |

> [!key-insight] tzdata is NOT compiled by zic
> CUBRID does **not** use the standard `zic` tool to compile tzdata into binary TZif files. Instead it has its own parser (`tz_compile.c`) that reads the same IANA text format and emits a C source file containing static arrays. This is the only shipped parser for this data.

## Runtime Data Structures (`timezone_lib_common.h`)

The generated `timezones.c` populates a `TZ_DATA` struct whose fields the engine reads at runtime:

```c
struct tz_data {
  int country_count;       TZ_COUNTRY *countries;
  int timezone_count;      TZ_TIMEZONE *timezones;
  char **timezone_names;
  int offset_rule_count;   TZ_OFFSET_RULE *offset_rules;  // gmt_off + DST ruleset
  int name_count;          TZ_NAME *names;                // canonical + alias names
  int ds_ruleset_count;    TZ_DS_RULESET *ds_rulesets;
  int ds_rule_count;       TZ_DS_RULE *ds_rules;          // from/to year, in_month, at_time
  int ds_leap_sec_count;   TZ_LEAP_SEC *ds_leap_sec;
  // Windows only:
  int windows_iana_map_count;  TZ_WINDOWS_IANA_MAP *windows_iana_map;
  char checksum[TZ_CHECKSUM_SIZE + 1];  // MD5 of tzdata content
};
```

Zone IDs are encoded as 10-bit values (`TZ_ZONE_ID_MAX = 0x3ff`), stored inside the packed `TZ_ID` and `TZ_REGION` types used in `DB_DATETIMETZ` and `DB_TIMESTAMPTZ` on-disk layouts.

## Runtime API (`tz_support.h`)

```c
int  tz_load(void);               // dlopen libcubrid_timezones.so, call tz_get_data()
void tz_unload(void);             // dlclose

// Conversion and normalization
int  tz_conv_tz_datetime_w_region(...);
int  tz_create_datetimetz(...);
int  tz_create_timestamptz(...);
int  tz_utc_datetimetz_to_local(...);
const TZ_DATA *tz_get_data(void); // pointer into loaded lib's static data
```

`tz_load()` is called early in `boot_restart_server()` (see [[components/server-boot|server-boot]]).

## Build Pipeline

See [[components/timezone-build|timezone-build]] for the full step-by-step description.

Summary:
1. **CMake** builds `gen_timezone` binary (links against `cubridsa`).
2. **CMake custom command** runs `gen_timezone tzdata/ tzlib/timezones.c` to emit the C source.
3. **CMake** compiles `timezones.c` into `cubrid_timezones` shared library with SOVERSION `MAJOR.MINOR`.
4. **Post-install** `make_tz.sh` allows operators to rebuild the library from updated tzdata without a full engine rebuild.

## Generation Modes

`make_tz.sh -g` and `cubrid gen_tz -g` accept:

| Mode | Behaviour |
|------|-----------|
| `new` | Parse all tzdata from scratch; generate new timezone ID arrays; wipes old encoding |
| `extend` | Keep existing timezone IDs intact; add new zones; update DST offsets; **migrates user DB data** |
| `update` | Refresh existing zone data only; no new zones added |

> [!warning] `extend` mode is destructive
> Running `make_tz.sh -g extend` iterates over every database listed in `$CUBRID_DATABASES/databases.txt` and upgrades stored `TIMESTAMPTZ` / `DATETIMETZ` values in place. Backup is mandatory before use.

## SQL Types Backed by This Module

| CUBRID Type | Parse-tree node (`PT_TYPE_*`) | Internal type |
|------------|-------------------------------|---------------|
| `TIMESTAMPTZ` | `PT_TYPE_TIMESTAMPTZ` | `DB_TIMESTAMPTZ` |
| `DATETIMETZ` | `PT_TYPE_DATETIMETZ` | `DB_DATETIMETZ` |
| `DATETIMELTZ` | `PT_TYPE_DATETIMELTZ` | `DB_DATETIME` (session-local) |
| `TIMESTAMPLTZ` | `PT_TYPE_TIMESTAMPLTZ` | `DB_UTIME` (session-local) |

The `LTZ` variants resolve to the session timezone at display time; the `TZ` variants encode the zone ID in the stored value.

## Related

- [[components/server-boot|server-boot]] — `tz_load()` called during server restart
- [[components/parse-tree|parse-tree]] — `PT_TYPE_DATETIMETZ`, `PT_TYPE_TIMESTAMPLTZ`, etc.
- [[components/timezone-build|timezone-build]] — detailed `make_tz.sh` pipeline
- [[sources/cubrid-timezones|cubrid-timezones]] — source summary
