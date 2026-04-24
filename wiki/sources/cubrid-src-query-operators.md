---
created: 2026-04-23
type: source
title: "CUBRID src/query/ — Operator & Evaluator Family"
date: 2026-04-23
source_type: codebase
files_read:
  - "src/query/arithmetic.h"
  - "src/query/arithmetic.c (first 300 lines + internal helpers)"
  - "src/query/numeric_opfunc.h"
  - "src/query/numeric_opfunc.c (first 150 lines)"
  - "src/query/string_opfunc.h"
  - "src/query/string_opfunc.c (first 200 lines)"
  - "src/query/crypt_opfunc.h"
  - "src/query/crypt_opfunc.c (first 200 lines)"
  - "src/query/string_regex.hpp"
  - "src/query/string_regex_constants.hpp"
  - "src/query/string_regex.cpp (full)"
  - "src/query/query_opfunc.h"
  - "src/query/query_opfunc.c (lines 1-300, 6000-7027)"
  - "src/query/query_evaluator.h"
  - "src/query/query_evaluator.c (lines 1-300, 800-1800)"
tags:
  - source
  - cubrid
  - query
  - operators
status: ingested
pages_created:
  - "[[components/query-arithmetic]]"
  - "[[components/query-numeric]]"
  - "[[components/query-string]]"
  - "[[components/query-regex]]"
  - "[[components/query-crypto]]"
  - "[[components/query-opfunc]]"
  - "[[components/query-evaluator]]"
---

# CUBRID `src/query/` — Operator & Evaluator Family

Source ingest covering the full operator evaluation stack: arithmetic, fixed-point numeric, string/datetime, regex, crypto, generic function dispatcher, and predicate evaluator.

## Summary

Seven component pages created from six C/C++ source files (~55K lines total). The operator layer forms a three-tier stack:

1. **Leaf implementations**: `arithmetic.c` (math/JSON), `numeric_opfunc.c` (NUMERIC type), `string_opfunc.c` (string/datetime), `crypt_opfunc.c` (MD5/SHA/AES), `string_regex.cpp` (REGEXP dispatch)
2. **Dispatcher**: `query_opfunc.c` — binary operators + function-code switch
3. **Predicate engine**: `query_evaluator.c` — PRED_EXPR tree walk with three-valued logic

## Key findings

- `string_opfunc.c` is client+server (not SERVER_MODE-gated) by design; individual functions are selectively gated.
- The regex backend is a server parameter (`PRM_ID_REGEXP_ENGINE`) chosen globally at compile time, not per-query.
- AND short-circuit fires only on V_FALSE, not V_UNKNOWN — correct SQL semantics but non-obvious.
- `cub_reg_traits::lookup_collatename` always throws — POSIX collatename syntax is intentionally disabled for consistency.
- `DB_NUMERIC` uses binary big-integer (not BCD); decimal ops require power-of-10 table lookups pre-computed at server boot.
- The `qdata_evaluate_function` switch in `query_opfunc.c` handles ~40 function codes including 22 JSON functions, 5 REGEXP variants, collection constructors, CONNECT_BY, and hierarchy functions.

## Pages Created

- [[components/query-arithmetic]] — scalar math + JSON built-ins
- [[components/query-numeric]] — DB_NUMERIC fixed-point engine
- [[components/query-string]] — string/datetime/LOB/TZ built-ins
- [[components/query-regex]] — REGEXP/RLIKE RE2 vs std::regex dispatch
- [[components/query-crypto]] — MD5/SHA/AES/CRC32 via OpenSSL
- [[components/query-opfunc]] — binary operators + function dispatcher
- [[components/query-evaluator]] — PRED_EXPR tree walk, three-valued logic
