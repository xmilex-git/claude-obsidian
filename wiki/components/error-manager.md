---
type: component
parent_module: "[[components/base|base]]"
path: "src/base/"
status: developing
purpose: "Thread-local error stack with severity levels, file/line capture, and message catalog formatting — used by every CUBRID module"
key_files:
  - "error_code.h (~1700 #define error codes, always negative)"
  - "error_manager.c/h (er_set, er_errid, er_msg, er_stack_push/pop, ASSERT_ERROR)"
  - "error_context.hpp/cpp (cuberr::context — C++ per-thread stackable error levels)"
tags:
  - component
  - cubrid
  - error-handling
related:
  - "[[components/base|base]]"
  - "[[Error Handling Convention]]"
  - "[[components/memory-alloc|memory-alloc]]"
  - "[[components/system-parameter|system-parameter]]"
  - "[[components/parallel-query|parallel-query]]"
created: 2026-04-23
updated: 2026-04-23
---

# `error_manager` — Error Handling Subsystem

Implementation of [[Error Handling Convention]] for the CUBRID engine. Provides a thread-local error stack, severity routing, message catalog formatting, and file/line capture. Used by every module in the engine.

## Core Concepts

### Error Codes (`error_code.h`)

All ~1700 error codes are `#define` negative integers:
```c
#define NO_ERROR                          0
#define ER_FAILED                        -1
#define ER_OUT_OF_VIRTUAL_MEMORY         -2
// ... ~1700 codes total
```

> [!warning] Six-place update rule
> Adding a new error code requires updates in six places: `error_code.h`, `dbi_compat.h`, `msg/en_US.utf8/cubrid.msg`, `msg/ko_KR.utf8/cubrid.msg`, `ER_LAST_ERROR`, and CCI's `base_error_code.h` (if client-facing). Skipping any location causes build or runtime failures.

### Severity Levels

```c
enum er_severity {
  ER_FATAL_ERROR_SEVERITY,    // highest — aborts process
  ER_ERROR_SEVERITY,          // standard error
  ER_SYNTAX_ERROR_SEVERITY,   // SQL syntax
  ER_WARNING_SEVERITY,        // warning, does not abort
  ER_NOTIFICATION_SEVERITY    // lowest — informational
};
```

Severity controls log routing and exit behavior. `ER_FATAL_ERROR_SEVERITY` can trigger process abort depending on `er_exit_ask` configuration.

### File/Line Capture

The `ARG_FILE_LINE` macro expands to `__FILE__, __LINE__` and is always the second argument to `er_set`:

```c
#define ARG_FILE_LINE   __FILE__, __LINE__

er_set(ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_HEAP_UNKNOWN_OBJECT, 3, oid->volid, oid->pageid, oid->slotid);
```

## Public API

### Setting Errors

```c
// Primary — variadic args match message format string
void er_set(int severity, const char *file, int line, int err_id, int num_args, ...);

// With OS errno appended to message
void er_set_with_oserror(int severity, const char *file, int line, int err_id, int num_args, ...);

// With FILE* for file-path context
void er_set_with_file(int severity, const char *file, int line, int err_id, FILE *fp, int num_args, ...);
```

### Reading Errors

```c
int         er_errid(void);         // current error code (NO_ERROR if none)
int         er_errid_if_has_error(void); // same, but asserts if NO_ERROR
int         er_get_severity(void);  // current severity
const char *er_msg(void);           // formatted message string
bool        er_has_error(void);     // true if er_errid() != NO_ERROR
void        er_clear(void);         // clear current error
```

### Error Stack (save/restore)

```c
void er_stack_push(void);              // save current error, start fresh
void er_stack_push_if_exists(void);    // only push if error exists
void er_stack_pop(void);               // restore saved error
void er_stack_pop_and_keep_error(void); // pop but keep current if set
void er_stack_clearall(void);          // clear entire stack
```

> [!key-insight] Stack usage pattern
> Code that calls subroutines which may set their own errors—but that error should not pollute the caller's error state—uses `er_stack_push()` before the call and `er_stack_pop()` after. This is critical in nested scan/fetch operations.

## Convenience Macros

### Simple set + assign

```c
// ERROR0..ERROR5: set warning + assign local variable
ERROR0(error, ER_SOME_CODE);
ERROR2(error, ER_SOME_CODE, arg1, arg2);

// ERROR_SET_WARNING / ERROR_SET_ERROR families
ERROR_SET_ERROR(error, ER_SOME_CODE);
ERROR_SET_ERROR_2ARGS(error, ER_SOME_CODE, arg1, arg2);
```

### Assertion helpers

```c
ASSERT_ERROR();               // assert er_errid() != NO_ERROR
ASSERT_ERROR_AND_SET(ec);    // assign er_errid() to ec, assert non-zero
ASSERT_NO_ERROR();            // assert er_errid() == NO_ERROR
```

```c
// Release-mode: logs ER_FAILED_ASSERTION instead of crashing
assert_release(expr);
assert_release_error(expr);  // ER_ERROR_SEVERITY variant
```

### Debug logging

```c
er_log_debug(ARG_FILE_LINE, "fmt %s %d", str, num);
// only fires if PRM_ID_ER_LOG_DEBUG is set — see [[components/system-parameter]]
```

## C++ Error Context (`error_context.hpp`)

`cuberr::context` wraps the thread-local error state as a C++ object with stackable levels. Used by the parallel query subsystem to move per-worker errors to the main thread:

```cpp
// In worker thread before exit:
move_top_error_message_to_this();  // snapshots error into err_messages_with_lock
```

See [[components/parallel-query|parallel-query]] — "Error propagation across threads".

## Initialization

```c
int  er_init(const char *msglog_filename, int exit_ask);
void er_final(ER_FINAL_CODE do_global_final);  // ER_THREAD_FINAL or ER_ALL_FINAL
```

Must be called once per process (server/client startup). Message catalog path configures where formatted strings are loaded from.

## Helper Macros for Common Error Classes

```c
ER_IS_LOCK_TIMEOUT_ERROR(err)    // ER_LK_UNILATERALLY_ABORTED + timeout variants
ER_IS_ABORTED_DUE_TO_DEADLOCK(err)
ER_IS_SERVER_DOWN_ERROR(err)     // network/connection failure codes
```

## Related

- [[Error Handling Convention]] — project-wide convention this implements
- [[components/base|base]] — parent component hub
- [[components/system-parameter|system-parameter]] — `PRM_ID_ER_LOG_DEBUG`, `PRM_ID_ER_LOG_LEVEL`
- [[components/parallel-query|parallel-query]] — cross-thread error propagation via `cuberr::context`
- Source: [[sources/cubrid-src-base|cubrid-src-base]]
