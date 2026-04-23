---
type: concept
title: "Code Style Conventions"
status: developing
tags:
  - concept
  - cubrid
  - style
  - convention
related:
  - "[[CUBRID]]"
  - "[[Memory Management Conventions]]"
  - "[[Error Handling Convention]]"
created: 2026-04-23
updated: 2026-04-23
---

# Code Style Conventions

CI-enforced. PRs that fail style are rejected.

## Formatting

| Aspect | Rule |
|--------|------|
| Indentation | 2 spaces, no tabs |
| Line width | 120 chars |
| C/H tooling | `indent -l120 -lc120` |
| C++/HPP tooling | `astyle --style=gnu` |
| Java tooling | `google-java-format` |
| Braces | GNU style — opening brace on new line, indented to body. Function braces at column 0. |
| Pointer asterisk | Attached to variable: `PT_NODE *node` |
| Calls | Space before `(` : `foo (x)` |

## Naming

- C functions: `module_action_object` — e.g., `pt_make_flat_name_list`, `qexec_execute_mainblock`
- C++ namespaces: short lowercase — `cubthread`, `lockfree`
- C++ classes: `snake_case`
- Macros / constants / C struct typedefs: `UPPER_SNAKE` — `NO_ERROR`, `PT_NODE`
- Header guards: `_FILENAME_H_` (NOT `#pragma once`)

## Includes

- Project headers: `"quotes"`
- System headers: `<angle brackets>`
- `config.h` first in `.c` files
- `memory_wrapper.hpp` MUST be the **last** include — see [[Memory Management Conventions]]
- C files: `/* ... */` only. C++ files: `//` allowed. File header: Apache 2.0 license block.

## PR title format

```
^\[[A-Z]+-\d+\]\s.+
```

Example: `[CBRD-12345] Fix buffer overflow in btree`. CLA required before merge.

## Related

- Source: [[cubrid-AGENTS]]
