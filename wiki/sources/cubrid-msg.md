---
type: source
title: "CUBRID msg/ — Localized Message Catalogs"
source_path: "msg/"
date_ingested: 2026-04-23
tags:
  - source
  - cubrid
  - i18n
  - error-handling
status: ingested
related:
  - "[[modules/msg|msg]]"
  - "[[components/message-catalog|message-catalog]]"
  - "[[components/error-manager|error-manager]]"
  - "[[Error Handling Convention]]"
---

# Source: `msg/` — Localized Message Catalogs

## What Was Read

| File | Lines read | Purpose |
|---|---|---|
| `msg/CMakeLists.txt` | all (113 lines) | Build pipeline: locale dirs, iconv, gencat |
| `msg/en_US.utf8/cubrid.msg` | first 100 | Format, set assignments, engine error strings |
| `msg/ko_KR.utf8/cubrid.msg` | first 100 | Korean translations, same structure |
| `msg/en_US.utf8/csql.msg` | first 40 | CSQL UI strings (set 1) |
| `msg/en_US.utf8/utils.msg` | first 118 | Utility string inventory (sets 1–59) |

## Key Findings

### Format: POSIX catgets (not gettext)

The files are POSIX message-catalog source format for `gencat` / `catgets(3)`. This is **not** GNU gettext `.po`/`.pot` — there are no `msgid`/`msgstr` pairs. Instead each message is identified by a `(set, msg-number)` integer pair, matched directly to error-code ordinals in C.

### Three catalogs, four locales

**Catalogs**: `cubrid` (engine + base), `csql` (REPL), `utils` (all admin utilities).

**Locales installed**: `en_US.utf8/`, `en_US/`, `ko_KR.utf8/`, `ko_KR.euckr/`. Only the two `.utf8` dirs are hand-maintained; the others are derived by the build (`cmake -E copy` or `iconv`).

### Build pipeline

`msg/CMakeLists.txt` defines `gen_msgs_<locale>` custom targets. For each of the four locales × three catalogs:

1. Copy or `iconv`-transcode the `.msg` source to the build dir.
2. Run `gencat` to produce the `.cat` binary.
3. Install both `.msg` and `.cat` under `${CUBRID_LOCALEDIR}/<locale>/`.

On Windows, a PowerShell script replaces `iconv` for the EUC-KR conversion.

### Engine error set (set 5 `MSGCAT_SET_ERROR`)

The `$set 5` block in `cubrid.msg` is the most important: each message number is the **absolute value of a negative error code** from `error_code.h`. Message 3 = `ER_OUT_OF_VIRTUAL_MEMORY` (`-2` → code 2, abs = 2; actual ordinal is 3 because codes start at 1 not 0 — the mapping is `abs(err_id)`). Format strings use positional `%N$type` arguments.

### Korean translations

`ko_KR.utf8/cubrid.msg` is a faithful translation of every string in `en_US.utf8/cubrid.msg`. The structure (set numbers, message numbers) is identical. Positional args let Korean sentences reorder subjects and objects relative to the English template.

### `utils.msg` covers ~35 admin utilities

Set numbers 1–59 are assigned to individual utilities (see the comment block at the top of `utils.msg`). New utilities add a new `$set` block; message numbering is independent per set.

## Pages Created / Updated

- Created: [[modules/msg|msg]] — full module page
- Created: [[components/message-catalog|message-catalog]] — format + C-side loader
- Updated: [[components/_index]] — added message-catalog entry
- Updated: [[sources/_index]] — added this source entry
