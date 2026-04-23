---
type: concept
title: "Error Handling Convention"
status: developing
tags:
  - concept
  - cubrid
  - error
  - convention
related:
  - "[[CUBRID]]"
  - "[[Memory Management Conventions]]"
  - "[[Code Style Conventions]]"
  - "[[components/base|base]]"
created: 2026-04-23
updated: 2026-04-23
---

# Error Handling Convention

C-style error model: error codes are negative integers, `NO_ERROR = 0`. **No C++ exceptions.**

## Setting an error

```c
er_set (ER_ERROR_SEVERITY, ARG_FILE_LINE, ER_CODE, ...);
```

## Checking

```c
if (error != NO_ERROR)
  {
    // handle
  }
```

## Adding a new error code — touches **6 files**

> [!key-insight] Six-place update for any new error code
> Forgetting one breaks the build, breaks i18n, or silently fails on the client.

1. `src/base/error_code.h` — define the constant
2. `src/compat/dbi_compat.h` — client-visible mirror
3. `msg/en_US.utf8/cubrid.msg` — English message
4. `msg/ko_KR.utf8/cubrid.msg` — Korean message
5. Update `ER_LAST_ERROR` constant
6. `cubrid-cci/base_error_code.h` — only if client-facing (CCI driver)

## Why it matters

Because there are no exceptions, **every callable** must propagate errors via return codes. Missed checks become silent corruption. The convention is consistent across `.c` and `.cpp` engine files.

## Related

- [[Memory Management Conventions]]
- [[modules/cubrid-cci|cubrid-cci]] (carries the client-facing copy of error codes)
- Source: [[cubrid-AGENTS]]
