---
type: component
parent_module: "[[modules/timezones|timezones]]"
path: "timezones/CMakeLists.txt, timezones/make_tz.sh, timezones/tzlib/build_tz.sh, src/executables/generate_timezone.cpp, src/base/tz_compile.h"
status: developing
purpose: "Two-phase build pipeline: IANA tzdata text files → gen_timezone → timezones.c → libcubrid_timezones shared library"
key_files:
  - "timezones/CMakeLists.txt (CMake targets: gen_timezone executable + cubrid_timezones shared lib)"
  - "timezones/make_tz.sh (post-install updater: invokes gen_tz + build_tz.sh)"
  - "timezones/tzlib/build_tz.sh (inner compiler: gcc -fPIC timezones.c → libcubrid_timezones.so)"
  - "src/executables/generate_timezone.cpp (gen_timezone main: calls timezone_compile_data)"
  - "src/base/tz_compile.h (SA_MODE compile-time API: timezone_compile_data)"
tags:
  - component
  - cubrid
  - timezones
  - build
  - codegen
related:
  - "[[modules/timezones|timezones]]"
  - "[[components/server-boot|server-boot]]"
  - "[[sources/cubrid-timezones|cubrid-timezones]]"
created: 2026-04-23
updated: 2026-04-23
---

# Timezone Build Pipeline

The `timezones/` build toolchain converts IANA timezone text files into `libcubrid_timezones.so` through two distinct phases: a codegen phase (text → C) and a compile phase (C → shared object). The two phases are designed to be run independently so operators can refresh timezone data post-deployment without a full engine rebuild.

## Phase 1 — Codegen: tzdata → timezones.c

### CMake path (initial build)

CMake defines a custom command in `timezones/CMakeLists.txt`:

```cmake
add_executable(gen_timezone
  ${EXECUTABLES_DIR}/generate_timezone.cpp)
target_link_libraries(gen_timezone LINK_PRIVATE cubridsa)
target_compile_definitions(gen_timezone PRIVATE SA_MODE)

add_custom_command(
  OUTPUT tzlib/timezones.c
  COMMAND gen_timezone ${CMAKE_CURRENT_SOURCE_DIR}/tzdata tzlib/timezones.c
)
```

Key points:
- `gen_timezone` links against `cubridsa` (the standalone library) and is compiled `SA_MODE` — this is what exposes `timezone_compile_data()` via `tz_compile.h`.
- `timezones.c` is tagged `PROPERTIES GENERATED true`; it is never committed to VCS.
- `cubrid_timezones` SHARED library depends on `gen_timezone` so CMake enforces the correct build order.

### generate_timezone.cpp entry point

```cpp
// src/executables/generate_timezone.cpp
int main(int argc, char **argv) {
  const char *tzdata_input_path  = argv[1];   // e.g. timezones/tzdata/
  const char *timezones_dot_c_output_path = argv[2]; // tzlib/timezones.c

  timezone_compile_data(tzdata_input_path,
                        TZ_GEN_TYPE_NEW,
                        NULL,                   // no database (batch-DB path for extend)
                        timezones_dot_c_output_path,
                        checksum_str);
}
```

`timezone_compile_data()` (in `src/base/tz_compile.c`) reads all IANA text files from the tzdata directory, parses Zone/Rule/Link records, computes an MD5 checksum of the data, and emits C static arrays for every `TZ_DATA` sub-array.

### Post-install / update path (make_tz.sh)

For operators updating timezone data after deployment, `make_tz.sh` (installed to `$CUBRID/bin/`) wraps the same codegen step using the engine's own `cubrid gen_tz` subcommand:

```sh
# new mode (default): rebuild from scratch
cubrid gen_tz -g new

# extend mode: preserve existing TZ IDs, add/update zones, migrate DB data
for DATABASE_NAME in $ALL_DATABASES; do
  cubrid gen_tz -g extend $DATABASE_NAME
done
```

`cubrid gen_tz` internally calls the same `timezone_compile_data()` path with `TZ_GEN_TYPE_EXTEND` rather than `TZ_GEN_TYPE_NEW`.

## Phase 2 — Compile: timezones.c → libcubrid_timezones.so

### CMake path

```cmake
add_library(cubrid_timezones SHARED tzlib/timezones.c)
set_target_properties(cubrid_timezones PROPERTIES
  SOVERSION "${CUBRID_MAJOR_VERSION}.${CUBRID_MINOR_VERSION}")
```

On Windows, the output is renamed to `libcubrid_timezones.dll`. On AIX, the static-archive wrapper `libcubrid_timezones.a` is used.

### Post-install path (build_tz.sh)

`timezones/tzlib/build_tz.sh` (installed to `$CUBRID/timezones/tzlib/`) replicates the CMake compile step as a plain shell script for post-deployment use:

```sh
# 64-bit release (default)
gcc -m64 -Wall -fPIC -c timezones.c
gcc -m64 -shared -o libcubrid_timezones.so timezones.o
rm -f timezones.o
```

Supports 32-bit (`-m32`) and debug (`-g`) flags. On AIX, uses `ar -X64 cru` to create the `.a` archive.

`make_tz.sh` then moves the resulting `.so` to `$CUBRID/lib/` and removes the intermediate `timezones.c`.

## Generation Mode Reference

`timezone_compile_data()` accepts a `TZ_GEN_TYPE` discriminant:

| Constant | Value | Behaviour |
|----------|-------|-----------|
| `TZ_GEN_TYPE_NEW` | 0 | Fresh parse; emits new timezone ID and name arrays; used at initial build |
| `TZ_GEN_TYPE_UPDATE` | 1 | Refreshes existing zone data (GMT offsets, DST rules); no new zones; no ID arrays emitted |
| `TZ_GEN_TYPE_EXTEND` | 2 | Like UPDATE but also adds new zones, preserves old IDs; **triggers per-database migration** |

## Install Targets

| Artifact | Destination | Platform |
|----------|-------------|----------|
| `libcubrid_timezones.so` | `$CUBRID_LIBDIR` | Linux/macOS |
| `libcubrid_timezones.dll` | `$CUBRID_LIBDIR` | Windows |
| `make_tz.sh` | `$CUBRID_BINDIR` | UNIX |
| `make_tz.bat` (from `make_tz_x64.bat`) | `$CUBRID_BINDIR` | Windows |
| `build_tz.sh` | `$CUBRID_TZDIR/tzlib` | UNIX |
| `tzdata/` directory | `$CUBRID_TZDIR` | All (excludes `Makefile.am`) |
| `timezone_lib_common.h` | `$CUBRID_TZDIR/tzlib` | All |

## Checksum and ABI Stability

`timezone_compile_data()` computes an MD5 checksum of the tzdata content and embeds it in the generated `TZ_DATA.checksum` field (32-char hex string). The engine can detect a library mismatch by comparing this checksum at `tz_load()` time.

Zone IDs are 10-bit integers (`TZ_ZONE_ID_MAX = 0x3ff`) stored inside `DB_DATETIMETZ` and `DB_TIMESTAMPTZ` on-disk values. If a `new`-mode rebuild changes ID assignments, stored TZ-typed column data becomes invalid — this is why `extend` mode exists and why a database migration pass is mandatory.

> [!warning] New-mode rebuild invalidates stored TZ data
> Running `make_tz.sh -g new` (or any `TZ_GEN_TYPE_NEW` codegen) reassigns all zone IDs. Any database containing `TIMESTAMPTZ` or `DATETIMETZ` columns will have corrupt timezone references after the new library is loaded. Always use `extend` mode when databases exist.

## Related

- [[modules/timezones|timezones]] — module overview, tzdata file inventory, SQL types
- [[components/server-boot|server-boot]] — `tz_load()` called during `boot_restart_server()`
- [[sources/cubrid-timezones|cubrid-timezones]] — source summary
