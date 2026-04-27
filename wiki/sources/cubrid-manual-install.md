---
created: 2026-04-27
type: source
title: "CUBRID Manual — Installation, Upgrade, Environment, Quick Start"
source_path: "/home/cubrid/cubrid-manual/en/install_upgrade.rst, install.rst, env.rst, upgrade.rst, uninstall.rst, quick_start.rst, start.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - install
  - upgrade
  - env
  - ports
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-config-params]]"
  - "[[sources/cubrid-manual-admin]]"
  - "[[sources/cubrid-manual-ha]]"
  - "[[components/executables]]"
  - "[[components/cub-server-main]]"
  - "[[Build Modes (SERVER SA CS)]]"
---

# CUBRID Manual — Installation, Upgrade, Environment

**Ingested:** 2026-04-27
**Source files:** `install_upgrade.rst` (17), `install.rst` (559), `env.rst` (476), `upgrade.rst` (399), `uninstall.rst` (87), `quick_start.rst` (94), `start.rst` (28) — 1660 lines total

## Section Map

| Section | File | Content |
|---|---|---|
| Install/upgrade hub | `install_upgrade.rst` | Toctree wrapper |
| Install (Linux + Windows) | `install.rst` | Platform support, RPM/SH/zip variants, post-install env, port firewall, CUBRID Service Tray |
| Environment variables | `env.rst` | `CUBRID`, `CUBRID_DATABASES`, `CUBRID_MSG_LANG`, `CUBRID_TMP`, language/charset, **port reference table** (Default, HA, Manager, PL) |
| Upgrade | `upgrade.rst` | Migration from any of 9.1/9.2/9.3/10.x/11.x and from 2008 R4.x; reserved word check; cubrid.conf/cubrid_broker.conf parameter renames; HA-aware migration |
| Uninstall | `uninstall.rst` | SH/RPM/Windows variants |
| Quick start | `quick_start.rst` | `cubrid service start`, `cubrid createdb testdb en_US` |
| Getting started | `start.rst` | Toctree (quick_start + qrytool) |

## Key Facts

### Platform & dependencies
- **64-bit Linux only** (since CUBRID 10.0 — 32-bit dropped). 64-bit Windows ≥7 (Windows 32-bit dropped after 10.1).
- **glibc ≥ 2.3.4** required.
- Required libs: ncurses (CUBRID ships against v5; `ncurses-compat-libs` may be needed on newer hosts), libgcrypt, libstdc++.
- **/etc/hosts** must map hostname ↔ IP correctly or DB server won't boot. (Yes, this is documented as a checklist item.)

### Install layouts
- **SH installer**: defaults to `~/CUBRID`. Creates `.cubrid.sh` / `.cubrid.csh` env-var script in user home.
- **RPM**: installs under `/opt/cubrid`, creates `cubrid` system user/group, registers `/etc/init.d/cubrid` script, **does NOT auto-create demodb** (run `/opt/cubrid/demo/make_cubrid_demo.sh` as cubrid user).
- **tar.gz / zip**: manual env-var setup; on Windows must install MS Visual C++ Redistributable.
- After SH install, demodb is auto-created in the install path.

### Environment variables (definitive list)
- **`CUBRID`** — install root. Required.
- **`CUBRID_DATABASES`** — location of `databases.txt` (the DB-registry file consulted by every utility).
- **`CUBRID_MSG_LANG`** — locale of utility/error messages. Values: `en_US`, `en_US.utf8`, `ko_KR.euckr`, `ko_KR.utf8`. Default `en_US` if unset. Restart required to apply.
- **`CUBRID_TMP`** — overrides temp file + UNIX-socket directory. Default Linux: `/tmp` (regular files), `$CUBRID/var/CUBRID_SOCK` (broker/PL UDS), `/tmp` (cub_master UDS).
  - **108-byte UDS path limit** — `CUBRID_TMP` value must keep total UDS path ≤ 108 chars (POSIX limit).
  - Must be **absolute path** (relative paths rejected).
  - Overrides `java.io.tmpdir` in `stored_procedure_vm_options` for `cub_pl`.
- **`CUBRID_LANG`** + **`CUBRID_CHARSET`** are **removed** (since 9.0). Charset/locale now locked at `cubrid createdb <db> <locale>` time.

### glibc memory tunables (commented in `.cubrid.sh`)
- `MALLOC_MMAP_MAX_` (default 65536)
- `MALLOC_MMAP_THRESHOLD_` (128 KiB)
- `MALLOC_TOP_PAD_` (0)
- `MALLOC_TRIM_THRESHOLD_` — **set to 0 by default** in installer's `.cubrid.sh` (only one not commented out). Prevents glibc from holding memory back from kernel.
- `MALLOC_ARENA_MAX` (8 × cores)
- `LD_PRELOAD=/usr/lib64/jemalloc.so.1` example commented out — explicit support for substituting the allocator.

### Linux kernel tuning
- **THP (Transparent Huge Pages) should be set to `never`** for production — manual explicitly recommends. Both `/sys/kernel/mm/transparent_hugepage/enabled` and `defrag` should be `never`.

### Locales (creation-time options for `cubrid createdb <db> <locale>`)
- Built-in: en_US (or en_US.iso88591), ko_KR.euckr, ko_KR.utf8, de_DE.utf8, es_ES.utf8, fr_FR.utf8, it_IT.utf8, ja_JP.utf8, km_KH.utf8, tr_TR.utf8, vi_VN.utf8, zh_CN.utf8, ro_RO.utf8.
- **Charset is permanent** — cannot be changed after `createdb`.

### Default port matrix (`env.rst:232-279`)
| Listener | Linux port | Windows port | Notes |
|---|---|---|---|
| `cub_broker` | `BROKER_PORT` (33000) | same | App connects here first |
| CAS | `BROKER_PORT` (Linux: same socket via `SCM_RIGHTS`) | `APPL_SERVER_PORT` … +N-1 | Windows allocates a port range per CAS |
| `cub_master` | `cubrid_port_id` (1523) | same | One per host |
| `cub_server` | `cubrid_port_id` (Linux: UDS auto-upgrade) | random ephemeral | Win firewall must allow `cub_server.exe` program-wide |
| Client ECHO | port 7 | port 7 | Keepalive — disable via `check_peer_alive=none` if blocked |
| Manager | 8001 (`cm_port`) | 8001 | |
| PL (`cub_pl`) | `stored_procedure_port` (default 0=random) | same | UDS available on Linux via `stored_procedure_uds=yes` |
| HA heartbeat | `ha_port_id` UDP | n/a (HA Linux-only) | master ↔ master |

### Linux specifics
- **TCP socket auto-upgrades to UNIX domain socket** for cub_master ↔ cub_server when both are local — confirms wiki's [[Architecture Overview]] claim.
- On Linux, the established TCP connection between broker and client is passed to the CAS via `SCM_RIGHTS` (no extra port needed). On Windows, broker tells client which CAS port to reconnect to.

### Upgrade
- DB volume format is **NOT compatible** between 11.3 and 11.4 (same family). Migration via `unloaddb` + `loaddb` is mandatory.
- Compatibility table: every minor version is its own island. Only adjacent within-major (e.g. 11.2 ↔ 11.3) is interoperable.
- **`check_reserved.sql`** script (shipped with installer + at `ftp.cubrid.org/CUBRID_Engine/11.4/`) detects identifiers that became reserved in the new version.
- **Standard upgrade path** (every from-version maps to this): stop service → run check_reserved → unloaddb → deletedb → install new → createdb (mind locale) → loaddb → backup → reconfigure cubrid.conf → start.
- **`log_buffer_size` minimum bumped** to 2 MB (was 48 KB / 3 pages of 16 KB) — old configs must increase.
- **`sort_buffer_size` capped at 2 GB**.
- **Parameter renames** (old → new, multiple unit changes): `lock_timeout_in_secs` → `lock_timeout` (msec), `checkpoint_every_npages` → `checkpoint_every_size` (byte), `checkpoint_interval_in_mins` → `checkpoint_interval` (msec), `max_flush_pages_per_second` → `max_flush_size_per_second` (byte), `sync_on_nflush` → `sync_on_flush_size` (byte), `sql_trace_slow_msecs` → `sql_trace_slow` (msec).
- **Removed** params: `single_byte_compare`, `intl_mbs_support`, `lock_timeout_message_type`, `SELECT_AUTO_COMMIT`.
- **Broker `KEEP_CONNECTION=OFF` is removed** — must be `ON` or `AUTO`.
- **`APPL_SERVER_MAX_SIZE_HARD_LIMIT` capped at 2,097,151** (≈ 2 GB), min 1024 MB.
- **`CCI_DEFAULT_AUTOCOMMIT` default changed to ON** somewhere in the 4.x line — old configs that relied on OFF must explicitly set OFF.
- HA migration steps (C7-C10): install on slave → restore master backup on slave → reconfigure ha_mode + ha_port_id + cubrid_ha.conf → start broker.

### Quick start
- `cubrid service start` boots master + broker + manager + (since 11.4) PL server (cub_pl).
- `cubrid createdb testdb en_US` creates with default 1.5 GB (data 512 MB + active log 512 MB + bg-archive log 512 MB).
- Auto-start: edit `[service]` block in `cubrid.conf` to `service=server,broker,manager` + `server=testdb`.

## Cross-References

- [[sources/cubrid-manual-config-params]] — full system parameter reference
- [[sources/cubrid-manual-admin]] — `cubrid createdb`, `cubrid addvoldb` flags
- [[sources/cubrid-manual-ha]] — HA migration scenario steps (C8 onwards)
- [[components/executables]] — implementation of `cubrid <cmd>` entry points
- [[components/cub-server-main]] · [[components/cub-master-main]] · [[components/broker-impl]] — process boot

## Incidental Wiki Enhancements

- [[components/cub-server-main]]: documented 108-byte UDS path limit and that local TCP connections auto-upgrade to UDS.
- [[components/broker-impl]]: documented `SCM_RIGHTS` socket-handoff (Linux) vs port-reconnect (Windows) split.
- [[Tech Stack]]: documented glibc memory tunable defaults and the manual's recommendation to disable THP.
- [[components/sp]]: documented that `stored_procedure_uds=yes` enables UDS for `cub_pl` IPC on Linux (default is TCP via `stored_procedure_port` regardless of OS).

## Key Insight

The install + upgrade chapters are stricter than they look — DB volume format breaks every minor (11.2 → 11.3 → 11.4 all need unload/load), the broker `KEEP_CONNECTION=OFF` removal is a silent-fail trap for old configs, and CUBRID_LANG/CUBRID_CHARSET removal means locale is permanently fixed at `createdb` time. Anyone planning a CUBRID upgrade should read `upgrade.rst` carefully *before* unloading.
