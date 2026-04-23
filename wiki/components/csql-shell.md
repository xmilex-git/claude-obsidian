---
type: component
parent_module: "[[modules/src|src]]"
path: "src/executables/csql*.c"
status: developing
purpose: "CSQL interactive SQL shell — argument parsing, DSO-based mode selection (SA/CS), REPL loop, session commands, result display"
key_files:
  - "csql_launcher.c (main() — arg parse, DSO load, dispatch)"
  - "csql.c (REPL engine: start_csql(), csql_execute_statements(), session commands)"
  - "csql_session.c (command buffer editing, history, multi-line accumulation)"
  - "csql_result.c (tabular result display, column width, pager integration)"
public_api:
  - "main() in csql_launcher.c"
  - "csql(argv0, csql_arg*) — exported symbol loaded at runtime from DSO"
  - "csql_get_message(index) — message string lookup, also loaded via DSO"
tags:
  - component
  - cubrid
  - csql
  - repl
  - shell
  - executables
related:
  - "[[components/executables|executables]]"
  - "[[components/connection|connection]]"
  - "[[Build Modes (SERVER SA CS)]]"
  - "[[modules/sa|sa module]]"
  - "[[modules/cs|cs module]]"
  - "[[sources/cubrid-src-executables|cubrid-src-executables]]"
created: 2026-04-23
updated: 2026-04-23
---

# `csql` — Interactive SQL Shell

`csql` is CUBRID's primary interactive SQL client, analogous to `psql` (PostgreSQL) or `mysql` (MySQL). It can connect to a running `cub_server` via CSS (CS_MODE, the default) or run in standalone mode with the database mounted in-process (SA_MODE, `-S` flag). The actual REPL engine (`csql()`) is loaded from a shared library at runtime, enabling the same launcher binary to switch modes.

## Architecture: DSO-Based Mode Switch

The `csql_launcher.c` `main()` does not contain any SQL engine logic. Instead it:

1. Parses all command-line options (28 options via `getopt_long`) into a `CSQL_ARGUMENT` struct.
2. Decides whether to load `libutil_sa` or `libutil_cs` based on `-S`/`-C` flags (SA default when no server active).
3. Calls `utility_load_library(LIB_UTIL_SA_NAME or LIB_UTIL_CS_NAME)` to `dlopen()` the appropriate shared library.
4. Resolves the symbol `csql` from that library and calls it.

This design means the `csql` binary itself is mode-agnostic. The link-time difference lives inside `libutil_sa` (links `cubridsa`, `SA_MODE`) vs `libutil_cs` (links `cubridcs`, `CS_MODE`).

```
csql_launcher.c:main()
  ├─ getopt_long() → CSQL_ARGUMENT
  ├─ utility_load_library(LIB_UTIL_SA_NAME / LIB_UTIL_CS_NAME)
  └─ csql(argv0, &csql_arg)   ← symbol from DSO → csql.c
         └─ start_csql(&csql_arg)
               ├─ db_login() / db_restart() — open DB connection
               ├─ readline loop (libedit on Linux/Mac, fallback on Windows)
               │     ├─ CSQL_SESSION_COMMAND_PREFIX (';' or '!') → csql_do_session_cmd()
               │     └─ SQL text → csql_edit_contents_append() → csql_execute_statements()
               └─ db_shutdown() / exit
```

## Command-Line Options (selected)

| Flag | Purpose |
|------|---------|
| `-S` / `--SA-mode` | Standalone — mount DB in-process (no server needed) |
| `-C` / `--CS-mode` | Client-server — connect to running `cub_server` |
| `-u user` | Database user |
| `-p password` | Password (avoid; use interactive prompt) |
| `-i file` | Read SQL from file (batch mode) |
| `-c "stmt"` | Execute SQL string and exit |
| `--no-auto-commit` | Disable auto-commit after each statement |
| `--read-only` | Open DB read-only |
| `--line-output` | One column per line (pipe-friendly) |
| `--plain-output` | No column separators |
| `--query-output` | CSV-like output with delimiter/enclosure options |
| `--sysadm` | System admin mode (bypass some auth) |
| `--skip-vacuum` | Skip deferred vacuum on connect |

## Session Commands (`;` prefix)

Session commands start with `;` or `!`. Key commands in `csql_do_session_cmd()`:

| Command | Effect |
|---------|--------|
| `;r` or `run` | Execute current command buffer |
| `;l` or `list` | Display command buffer with line numbers |
| `;e` or `edit` | Launch `$EDITOR` (vi default) on buffer |
| `;read file` | Read SQL from file into buffer |
| `;write file` | Write buffer to file |
| `;cd dir` | Change working directory |
| `;set param value` | Set system parameter at runtime |
| `;get param` | Get system parameter value |
| `;trace on/off` | Toggle query trace output |
| `;info` | Show schema info |
| `;connect db [host]` | Re-connect to a different database |
| `;exit` | Exit csql |

## REPL Internals

**Input accumulation.** Multi-line SQL is accumulated in a command buffer (`csql_edit_contents_*` in `csql_session.c`). The buffer is flushed on `;` (standalone session command) or when the buffer contains a complete SQL statement ending with `;`.

**readline/libedit.** On Linux/Mac, `csql.c` includes `<editline/readline.h>`. Keyword completion via `rl_attempted_completion_function` is compiled in but guarded by `ENABLE_UNUSED_FUNCTION` (currently inactive — completion stubs exist but are not wired). History is available through the standard `readline` history API.

**Custom prompt.** The `CUBRID_CSQL_PROMPT` environment variable controls the prompt format. Escape sequences: `\u`/`\U` → username, `\d`/`\D` → database name, `\h`/`\H` → hostname. Default prompt is `csql> `.

**Error handling.** Fatal errors use `setjmp`/`longjmp` via `csql_Exit_env` / `csql_Exit_status` rather than `exit()` — this ensures cleanup runs even on Windows where `exit()` from a DSO is problematic.

**Signal handling.** `SIGPIPE` is caught per-output-operation via `csql_Jmp_buf` (for pager interaction). `SIGINT` sets `csql_Is_sigint_caught` which the REPL loop checks after each readline call.

## Result Display (`csql_result.c`)

Results are printed in tabular format (column names, separator lines, rows). Column width can be constrained per-column via `csql_column_width_info_list`. Output is piped through `csql_Pager_cmd` (default `more`; configurable) unless `--no-pager` is set. Timing (`--time`) appends elapsed time after each query.

## Global State

| Variable | Purpose |
|----------|---------|
| `csql_Db_name[512]` | Currently connected database |
| `csql_Database_connected` | True when `db_restart()` succeeded |
| `csql_Is_interactive` | False in batch (`-i`/`-c`) mode |
| `csql_Row_count` | Rows returned by last SELECT |
| `csql_Num_failures` | Accumulated error count in batch mode |
| `csql_Is_echo_on` | Echo SQL before executing |
| `csql_Is_time_on` | Print timing (default on) |
| `csql_Is_histo_on` | Network histogram (off by default) |

## Related

- [[components/executables|executables]] — binary inventory hub
- [[Build Modes (SERVER SA CS)]] — how SA vs CS mode is chosen at link time
- [[modules/sa|sa module]] / [[modules/cs|cs module]] — the CMake targets that produce `libutil_sa` / `libutil_cs`
- [[components/connection|connection]] — CSS protocol that CS-mode csql uses to reach `cub_server`
