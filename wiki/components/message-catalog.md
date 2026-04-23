---
type: component
parent_module: "[[modules/msg|msg]]"
path: "msg/"
status: developing
purpose: "POSIX catgets-based message catalog format, loader lifecycle, and C API (msgcat_*) used by error-manager and all CUBRID tools to surface localized strings"
key_files:
  - "msg/en_US.utf8/cubrid.msg (engine error strings, sets 1/4/5+)"
  - "msg/en_US.utf8/csql.msg (CSQL UI strings, set 1)"
  - "msg/en_US.utf8/utils.msg (admin utility strings, sets 1–59)"
  - "msg/ko_KR.utf8/cubrid.msg (Korean translations)"
  - "msg/CMakeLists.txt (gencat + iconv pipeline)"
  - "src/base/message_catalog.c/h (runtime loader)"
tags:
  - component
  - cubrid
  - i18n
  - error-handling
related:
  - "[[modules/msg|msg]]"
  - "[[components/error-manager|error-manager]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[Error Handling Convention]]"
  - "[[components/dbi-compat|dbi-compat]]"
created: 2026-04-23
updated: 2026-04-23
---

# `message-catalog` — Localized Message Catalog

CUBRID uses **POSIX `catgets`-style** message catalogs for all user-facing strings: engine error messages, CSQL UI text, and every admin utility. The system has two halves: the `.msg` source files in `msg/` and the runtime loader in `src/base/message_catalog.c`.

## File Format

### Source `.msg` syntax

Files follow POSIX `catgets` message-source format:

```
$ lines starting with $ are comments

$set <N> <SYMBOLIC_NAME>
<msg-num> format string
```

- `$set N` starts a new set (group). The symbolic name is a C macro name used by callers.
- Lines without a `$` prefix are message entries: an integer message number followed by a space and then the message text through end-of-line. Backslash-newline continues a long message.
- Arguments use POSIX **positional** format specifiers: `%1$s`, `%2$d`, `%3$lld`, `%4$zu`, etc. Positional specifiers allow Korean translations to reorder arguments without changing C call sites.

### Set assignments in `cubrid.msg`

| Set | Macro | Contents |
|---|---|---|
| 1 | `MSGCAT_SET_GENERAL` | Generic argument-parsing diagnostics, copyright strings |
| 4 | `MSGCAT_SET_PARAMETER` | System-parameter read/write errors, config-file problems |
| 5 | `MSGCAT_SET_ERROR` | All ~1700 engine error messages (ordinal = `abs(error_code)`) |

Additional sets exist in `csql.msg` (set 1: `MSGCAT_CSQL_SET_CSQL`) and `utils.msg` (sets 1–59, one per admin utility).

### Compiled binary format

`gencat` compiles each `.msg` file into a binary `.cat` file. The build target `gen_msgs_<locale>` calls `gencat` for every locale × catalog combination. Both `.msg` and `.cat` files are installed to `${CUBRID_LOCALEDIR}/<locale>/`.

## Runtime Loader (`message_catalog.c`)

### Catalog handles

```c
// Catalog IDs (index into the open catalog array)
#define MSGCAT_CATALOG_CUBRID   0   // cubrid.msg/.cat
#define MSGCAT_CATALOG_CSQL     1   // csql.msg/.cat
#define MSGCAT_CATALOG_UTILS    2   // utils.msg/.cat
```

### Lifecycle

```c
// Open — called during er_init() for CUBRID, or at tool startup
int  msgcat_init(void);
void msgcat_final(void);

// Lookup — returns a C string (pointer into catalog memory; do not free)
char *msgcat_message(int catalog_id, int set_id, int msg_id);
```

`msgcat_init` locates the catalog directory from the `CUBRID_MSG` environment variable or a compiled-in default (`${CUBRID}/msg/<locale>/`). It opens the `.cat` binary with `catopen(3)`.

`msgcat_message` calls `catgets(3)` and returns the result. If the message is not found, the fallback text (from the same `catgets` call) is `"Missing message for error code N."` — set 5 message 1 in `cubrid.msg`.

### Integration with `er_set`

[[components/error-manager|error-manager]] calls the catalog at two points:

1. **Format string lookup**: `msgcat_message(MSGCAT_CATALOG_CUBRID, MSGCAT_SET_ERROR, abs(err_id))` retrieves the format string.
2. **Formatting**: `er_vsprintf` (or equivalent) applies the variadic args to the format string.

The formatted result is stored in the thread-local error buffer and returned by `er_msg()`.

## Locale Selection

The loader opens catalogs in the locale indicated by `LANG` / `CUBRID_MSG` at startup. In practice:

| Build output | Encoding | How produced |
|---|---|---|
| `en_US/` | UTF-8 | Copied verbatim from `en_US.utf8/` |
| `en_US.utf8/` | UTF-8 | Canonical source |
| `ko_KR.utf8/` | UTF-8 | Canonical Korean source |
| `ko_KR.euckr/` | EUC-KR | `iconv -f utf-8 -t euckr` of `ko_KR.utf8/` |

All four locale dirs are installed; only `en_US.utf8/` and `ko_KR.utf8/` require manual maintenance.

## Positional Arguments — Why They Matter

Standard `printf` format `%s %d` is order-dependent. Korean sentences have different word order from English. POSIX `%1$s %2$d` lets the same C `er_set` call produce correctly ordered output in both languages without any additional C code.

Example (`cubrid.msg` set 5, message 11):

```
EN: Unable to mount disk volume "%1$s". The database "%2$s", to which the disk
    volume belongs, is in use by user %3$s on process %4$d of host %5$s since %6$s.

KO: "%1$s" 디스크 볼륨을 마운트할 수 없습니다. 이 디스크 볼륨이 속한 데이터베이스
    "%2$s"은(는) 사용자 %3$s(프로세스 ID:%4$d, 호스트 %5$s)이(가) %6$s 부터 사용하고 있습니다.
```

Both resolve from the same `er_set(..., 6, volname, dbname, user, pid, host, since)` call.

## Six-Place Rule

> [!warning]
> Adding a new error code requires `msg/en_US.utf8/cubrid.msg` **and** `msg/ko_KR.utf8/cubrid.msg` to be updated simultaneously (places 3 and 4 of the [[Error Handling Convention]]). A missing entry silently degrades to "Missing message for error code N" at runtime and does not cause a build failure.

## Related

- [[modules/msg|msg]] — parent module; locale directory inventory, build pipeline
- [[components/error-manager|error-manager]] — `er_set` / `er_msg`; primary consumer of `MSGCAT_CATALOG_CUBRID`
- [[Error Handling Convention]] — six-place new-error-code rule
- [[components/dbi-compat|dbi-compat]] — error-code mirror; if out of sync with `cubrid.msg`, messages display incorrectly on client side
- [[components/csql-shell|csql-shell]] — consumes `MSGCAT_CATALOG_CSQL`
- [[components/utility-binaries|utility-binaries]] — consumes `MSGCAT_CATALOG_UTILS`
- [[components/system-parameter|system-parameter]] — parameter error strings in `cubrid.msg` set 4
- Source: [[sources/cubrid-msg|cubrid-msg]]
