---
type: source
title: "CUBRID locales/ — Locale Data + Build Toolchain"
source_type: directory
source_path: ".raw/cubrid/locales/"
ingested: 2026-04-23
status: summarized
tags:
  - source
  - cubrid
  - locale
  - i18n
related:
  - "[[modules/locales]]"
  - "[[modules/timezones]]"
  - "[[modules/msg]]"
created: 2026-04-23
updated: 2026-04-23
---

# Source: `locales/`

CUBRID's locale and collation data, plus the toolchain that compiles per-locale shared libraries the engine loads at boot.

## Read scope
- `locales/CMakeLists.txt`
- `locales/make_locale.sh` (driver script)
- `locales/data/ldml/` (LDML XML inventory)
- `locales/data/codepages/` (codepage tables inventory)
- `locales/loclib/build_locale.sh` reference

## Highlights
- **LDML-based** (Unicode CLDR-style XML) — not POSIX `localedef` format. Each locale file is a `cubrid_<lang>_<COUNTRY>.xml`.
- 11 locales ship by default: de_DE, es_ES, fr_FR, it_IT, ja_JP, km_KH, ko_KR, ro_RO, tr_TR, vi_VN, zh_CN. en_US is always available.
- Codepage tables for legacy single-byte (8859 family, CP1250/1258) and East-Asian DBCS (CP932/936/949).
- Collation drives ORDER BY / GROUP BY / B-tree comparisons, so a wrong locale lib at runtime corrupts index ordering.

## Pages produced
- [[modules/locales|locales (module page)]]

## Cross-cuts
- Loaded at boot by [[components/server-boot|server-boot]].
- Selected via `intl_collation` ([[components/system-parameter|system-parameter]]).
- Used by [[components/parser|parser]] (COLLATE clause) and [[components/btree|btree]] (key comparison).

## Follow-ups
- The `gen_locale` utility (engine binary) referenced by `build_locale.sh` is in `src/executables/` — worth a paragraph in [[components/utility-binaries]] noting locale generation.
- Collation algorithm details (`ducet.txt` consumption, tailoring rules from LDML) live in `src/base/locale_support.c` (or similar). Deeper ingest could file a `wiki/components/collation-engine.md` page.

## Source location
`.raw/cubrid/locales/` (symlink → `/Users/song/DEV/cubrid/locales/`).
