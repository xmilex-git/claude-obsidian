---
type: pr
pr_number: 7102
pr_url: "https://github.com/CUBRID/cubrid/pull/7102"
repo: "CUBRID/cubrid"
state: MERGED
is_draft: false
author: "youngjinj"
created_at: 2026-04-24
merged_at: 2026-05-08
closed_at:
merge_commit: "05a7befd8b714811632a16a97d3683ab3b397a0f"
base_ref: "develop"
head_ref: "CBRD-26744"
base_sha: "5e12a293c609a5d99c39b4c81a00b89b9ef91662"
head_sha: "05a7befd8b714811632a16a97d3683ab3b397a0f"
jira: "CBRD-26744"
files_changed:
  - "src/base/intl_support.c"
  - "src/broker/cas_common_function.c"
  - "src/broker/cas_execute.c"
  - "src/compat/db_macro.c"
  - "src/compat/dbtype.h"
  - "src/compat/dbtype_function.h"
  - "src/compat/dbtype_function.i"
  - "src/executables/csql.c"
  - "src/executables/csql_result_format.c"
  - "src/loaddb/load_db_value_converter.cpp"
  - "src/loaddb/load_sa_loader.cpp"
  - "src/method/method_query_util.cpp"
  - "src/object/authenticate_access_auth.cpp"
  - "src/query/string_opfunc.c"
related_components:
  - "[[components/db-value]]"
  - "[[components/cas]]"
  - "[[components/base]]"
  - "[[components/loaddb-executor]]"
  - "[[components/loaddb]]"
  - "[[components/csql-shell]]"
  - "[[components/authenticate]]"
related_sources:
  - "[[sources/cubrid-src-base]]"
  - "[[sources/cubrid-src-compat]]"
  - "[[sources/cubrid-src-broker]]"
ingest_case: "c"
triggered_baseline_bump: true
baseline_before: "5e12a293c609a5d99c39b4c81a00b89b9ef91662"
baseline_after: "05a7befd8b714811632a16a97d3683ab3b397a0f"
reconciliation_applied: true
reconciliation_applied_at: 2026-05-08
incidental_enhancements_count: 4
tags:
  - pr
  - cubrid
  - db-value
  - utf8
  - swar
  - performance
  - cleanup
created: 2026-05-08
updated: 2026-05-08
status: merged
---

# PR-7102-db-get-char-intl-cleanup

> [!info] PR metadata
> **Repo:** `CUBRID/cubrid` · **State:** `MERGED` · **Author:** `@youngjinj` · **Merge commit:** `05a7befd8`
> **Base → Head:** `develop` (`5e12a293c`) → `CBRD-26744` (`05a7befd8`)

> [!note] Ingest classification: case (c)
> Merge commit `05a7befd8` is a direct descendant of the just-bumped baseline `5e12a293c` on `develop`, with one intermediate commit (`f3d6434d` = PR #7145, `.travis.yml` removal — no wiki coverage). PR-reconciliation applied; baseline bumped to `05a7befd8` (transitively absorbing PR #7145).

## Summary

Two-pronged cleanup of UTF-8 character counting in the CAS / `compat` / loaddb hot paths.

1. **API simplification.** `db_get_char(DB_VALUE *)` loses its `int *length` out-parameter — 9 of its 11 call sites were already discarding the value into a `dummy` int. Pre-PR every `dbval_to_net_buf` VARCHAR fetch ran an unconditional `intl_char_count` (O(N) UTF-8 scan) just to populate that ignored value.
2. **Algorithmic improvement of the remaining counters.** `intl_count_utf8_chars`, `intl_count_utf8_bytes`, and `intl_check_utf8` gain an 8-byte SWAR ASCII fast path: while the next 8 bytes have no high bit set, they're all ASCII and contribute exactly 8 chars / 8 bytes / 0 invalid sequences — skip the per-byte work. Bulk-ASCII payloads (the common case for English text and identifiers) now run at ~8× throughput.

Concurrent micro-optimizations: same-TU `intl_nextchar_utf8(s, &n)` calls are replaced with direct `intl_Len_utf8_char[*s]` table lookups (the table is the lookup `intl_nextchar_utf8` performs anyway, so the wrapper is pure overhead inside `intl_support.c`); precision-truncate paths in `db_string_truncate` / loaddb `ldr_str_db_char` / `ldr_str_db_varchar` / `to_db_generic_char` skip the char-count scan entirely when `byte_size <= precision` (because `char_count <= byte_count` for every CUBRID codeset, byte ≤ precision implies char ≤ precision); `DB_GET_STRING_PRECISION` macro deleted as dead.

## Motivation

CBRD-26744. The CAS `dbval_to_net_buf` path runs once per DB_TYPE_VARCHAR / DB_TYPE_CHAR cell on every fetch reply. Pre-PR the inlined `db_get_char(val, &dummy)` call did a full `intl_char_count` UTF-8 scan inside the accessor, then the caller threw the result away. For wide result sets this is the dominant CPU cost in CAS reply assembly; profiles attributed it to "string conversion" without the obvious diagnosis that the work was wholly unused.

The SWAR ASCII fast path is independent: even after removing the wasted call sites, the legitimately-required scans (precision enforcement, `intl_check_utf8` validation in `netval_to_dbval`) still walked every byte one at a time. The 8-byte word scan is a textbook trick — a `(word & 0x8080808080808080) == 0` test rejects any byte with MSB set (i.e. any non-ASCII byte) in one cycle.

## Changes

### Structural

- **API change (header-public):** `db_get_char (const DB_VALUE *)` — signature drops the `int *length` out-param. Declaration in `src/compat/dbtype_function.h`; inline body in `dbtype_function.i`. Macro alias `DB_GET_CHAR(v, l)` becomes `DB_GET_CHAR(v)`. **Source-level breaking change** for any out-of-tree caller; ABI moot because both old and new callers compile against the inline.
- **Removed macro:** `DB_GET_STRING_PRECISION(v)` — duplicate definition (existed in both `dbtype.h` and `dbtype_function.h`); no remaining call sites.
- **`intl_count_utf8_chars`** (`src/base/intl_support.c`) — head rewritten with SWAR ASCII fast path; falls through `goto slow_path:` to original loop only if a non-ASCII byte is seen. Slow path now uses `s += intl_Len_utf8_char[*s]` directly (was `s = intl_nextchar_utf8(s, &dummy)`). Net +91 / -3 LOC for the function.
- **`intl_count_utf8_bytes`** — same SWAR pattern. Slow path uses `byte_count += intl_Len_utf8_char[s[byte_count]]` (no `intl_nextchar_utf8` call).
- **`intl_check_utf8`** — adds an 8-byte ASCII skip *inside* the existing per-byte validation loop. Cannot fully fast-path because validation must inspect every multi-byte sequence; the inserted `while (p+8 <= p_end)` block accelerates the ASCII stretches between multi-byte clusters.
- **`intl_next_char` / `intl_next_char_pseudo_kor`** — UTF-8 branch becomes inline `*current_char_size = intl_Len_utf8_char[*s]; return s + *current_char_size;` (was a function call). External callers of `intl_next_char` see no API change.
- **`intl_reverse_string`** — same inlining of the UTF-8 lead-byte length lookup.
- **`intl_identifier_lower_string_size` / `intl_identifier_upper_string_size`** — drop unused `int src_len` + the `intl_char_count` call that populated it. The result was never read (the function returns the byte size of the lowercased/uppercased buffer, not a char count).

### Per-file notes

- `src/base/intl_support.c` (+105 / −14) — SWAR fast paths in 3 entry points, table-lookup inlining throughout the TU, dead `src_len` removal in 2 identifier-case-fold functions. ([[components/base]])
- `src/broker/cas_common_function.c` (+3 / −1) — `cas_common_bind_value_print` now passes `val_size - 1` to `intl_char_count` (excluding trailing NUL) and uses the out-arg directly instead of subtracting 1 from the return. Adds debug `assert (str_val[val_size - 1] == '\0')` documenting the CAS payload invariant. ([[components/cas]])
- `src/broker/cas_execute.c` (+7 / −8) — `netval_to_dbval` adds the same trailing-NUL `assert`, removes the discarded `intl_char_count` call before `db_make_char`. `dbval_to_net_buf` switches `DB_TYPE_CHAR` and `DB_TYPE_BLOB`/`CLOB` paths to the 1-arg `db_get_char`. `convert_db_value_to_string` likewise. ([[components/cas]])
- `src/compat/db_macro.c` (+22 / −17) — `db_string_truncate` `DB_TYPE_CHAR` branch reorders: byte-size check first (cheap), then `intl_char_count` only if `byte_size > precision`. Avoids a UTF-8 scan on every CHAR truncation request when no truncation is needed. ([[components/db-value]])
- `src/compat/dbtype.h` (−3) — `DB_GET_STRING_PRECISION` macro removed.
- `src/compat/dbtype_function.h` / `.i` (+8 / −9 combined) — `db_get_char` signature change; declaration, inline body, and `DB_GET_CHAR(v, l)` alias updated. The inline body also drops the `intl_char_count` calls that populated `*length` for `SMALL_STRING` and `MEDIUM_STRING` styles. ([[components/db-value]])
- `src/executables/csql.c` (+2 / −2) — `csql_display_trace` updated to 1-arg call; drops `dummy` local. ([[components/csql-shell]])
- `src/executables/csql_result_format.c` (+2 / −2) — same in `csql_db_value_as_string`.
- `src/loaddb/load_db_value_converter.cpp` (+27 / −24) — `to_db_generic_char` reorders precision check to byte-size first. **Bug fix bundled in:** `db_value_domain_init (val, type, char_count, 0)` is changed to `db_value_domain_init (val, type, precision, 0)`. The pre-PR code initialized the value's domain precision to the actual character count of the input, not the column's defined precision — a semantic misuse that diverged the loaded value's domain from the schema. (Surfaced during the `intl_char_count` removal sweep — only visible because `char_count` became locally scoped.) ([[components/loaddb-executor]])
- `src/loaddb/load_sa_loader.cpp` (+68 / −62) — `ldr_str_db_char` and `ldr_str_db_varchar` get the same byte-first reorder and the same `val.domain.char_info.length = precision` correction (was `= char_count`). ([[components/loaddb]])
- `src/method/method_query_util.cpp` (+2 / −2) — 1-arg `db_get_char` migration in `cubmethod::convert_db_value_to_string`. (Preexisting `std::string(nullptr)` UB on the error path was flagged by greptile and explicitly declined as out-of-scope by the author.)
- `src/object/authenticate_access_auth.cpp` (+5 / −6) — three `db_get_char` migrations in `au_object_revoke_all_privileges`, `au_user_revoke_all_privileges`, `update_auth_for_new_owner`. Drops 3 `len` locals. ([[components/authenticate]])
- `src/query/string_opfunc.c` (+3 / −2) — `db_inet_aton` 1-arg migration; replaces the discarded `cnt = char_count` with `cnt = db_get_string_size (string)` (byte size, which is what the rest of the function actually wanted — IP-string parsing is byte-oriented).

### Behavioral

- **Per-fetch CPU savings.** Removing `intl_char_count` from `db_get_char` eliminates an O(N) UTF-8 scan from every `dbval_to_net_buf` invocation on `DB_TYPE_VARCHAR` / `DB_TYPE_CHAR`. For ASCII workloads the SWAR fast path means the *remaining* counters run at ~8× throughput on the bulk of every payload, decohering only at non-ASCII characters.
- **Precision-truncate fast path.** `byte_size <= precision` now short-circuits `intl_char_count` in `db_string_truncate`, `to_db_generic_char`, `ldr_str_db_char`, `ldr_str_db_varchar`. Correctness rests on the codeset invariant `char_count(s) <= byte_count(s)` — true for UTF-8, EUC-KR, and ISO-8859 variants used by CUBRID, but **not** true in general (e.g. UTF-16 has 2 bytes per BMP char). The codebase exclusively uses byte-encoded codesets; the comment in each call site states the rationale.
- **loaddb domain-precision correction (semi-bug-fix).** Pre-PR, `val.domain.char_info.length` was set to `char_count` of the *input string* — meaning every loaded row's value had a precision matching its content rather than the column's schema precision. Post-PR, `length = precision` (the domain's defined precision). This affects:
  - `db_value_clone` / serialization paths that read `domain.char_info.length` will now see the schema precision.
  - Equality / comparison ops that compute on precision will see consistent values across rows.
  - The pre-PR behavior was tolerated because most consumers re-derive precision from the actual data, but it was inconsistent with `db_value_domain_init` semantics elsewhere in the engine.
- **CAS protocol assertion (debug only).** Two new asserts in `cas_common_function.c` and `cas_execute.c` document that string payloads from the wire arrive with `val_size - 1` data bytes plus a trailing `'\0'`. Pre-PR this was implicit knowledge in subtractions like `val_size--`; the asserts now state the invariant explicitly. Release builds unaffected.
- **No on-disk format change. No XASL change. No protocol wire change.**

### New surface (no existing wiki reference)

- `intl_count_utf8_chars`, `intl_count_utf8_bytes`, `intl_check_utf8` — the SWAR fast path is documented inline; not currently a dedicated wiki page.
- `intl_Len_utf8_char[]` lookup table — referenced by name only inside `intl_support.c`; baseline wiki has no coverage.
- `cas_common_function.c::cas_common_bind_value_print` — no dedicated wiki coverage.

## Review discussion highlights

- **`std::string(nullptr)` in `method_query_util.cpp::convert_db_value_to_string`** (greptile P1). On the `db_value_coerce` error path `val_str` stays `nullptr` and the `return std::string(val_str)` triggers UB. Author declined as out-of-scope for this PR — preexisting bug, separate ticket warranted. Worth flagging here; **not** filed as wiki incidental because no existing wiki page documents this function.
- **Truncate fast-path correctness** (greptile P2 → confirmed). Reviewer asked when the `byte_size <= precision` short-circuit accumulates wins vs. always-call. Author landed the optimization in commit `c603bf4` after explicit confirmation that it applies to every codeset CUBRID supports. Greptile ack: "consistent application across `db_macro.c`, `load_sa_loader.cpp`, `load_db_value_converter.cpp` and the `val.domain.char_info.length` semantic correction is clean."
- **No invariant changes to `intl_check_utf8` validation.** The SWAR addition only skips bytes the slow path would have classified as ASCII and incremented past; on any high-bit byte it falls through into the original validator without state change. The unit test surface for malformed-UTF-8 detection therefore stays valid.

## Reconciliation Plan

Applied during this ingest — see Pages Reconciled below.

## Pages Reconciled

- *(none — pre-PR wiki had no signature claims for `db_get_char`, no SWAR claims for `intl_count_utf8_*`, and no precision-init claims for the loaddb converters that needed updating)*

The four updates that *did* land are knowledge-additions to baseline-truth content, listed under Incidental wiki enhancements below — they were not contradictions of existing claims.

## Incidental wiki enhancements

- [[components/db-value]] — added `db_get_char (const DB_VALUE *) → DB_CONST_C_CHAR` to the `db_get_*` accessor table; previously only `db_get_string` was listed. Added a one-paragraph note that `db_get_char` returns the underlying `DB_CHAR.{sm,medium}.buf` directly without any character counting, so callers who need character count must call `intl_char_count` themselves. `[!gap]` filled.
- [[components/base]] — added a paragraph under "Internationalization" describing the `intl_count_utf8_chars` / `intl_count_utf8_bytes` / `intl_check_utf8` SWAR ASCII fast path and the `intl_Len_utf8_char[256]` lead-byte length table that drives all UTF-8 stepping in `intl_support.c`. `[!gap]` filled — baseline wiki had only one line on intl_support.
- [[components/cas]] — added a `[!key-insight]` callout under "SQL Execution Path" stating the CAS string-payload invariant: wire string values arrive with `val_size - 1` data bytes plus a trailing `'\0'`. This invariant is the basis for the `value[val_size - 1] == '\0'` asserts and for the `val_size--` adjustment before `intl_check_string`.
- [[components/loaddb-executor]] — added a paragraph to "DB_VALUE converter" noting that `to_db_generic_char` initializes the value's domain via `db_value_domain_init(val, type, precision, 0)` — using the column's *schema* precision, not the input's char count. Pre-PR code used `char_count`; this was reclassified as a latent inconsistency and corrected by PR #7102. `[!gap]` filled.

## Baseline impact

- Before: `5e12a293c609a5d99c39b4c81a00b89b9ef91662`
- After: `05a7befd8b714811632a16a97d3683ab3b397a0f`
- Bump triggered: `true` (transitively also absorbs PR #7145 — `.travis.yml` removal, no wiki impact)
- Logged: [[log]] under `[2026-05-08] baseline-bump | 5e12a293 → 05a7befd`

## Related

- [[prs/_index|PRs]]
- CUBRID upstream PR: https://github.com/CUBRID/cubrid/pull/7102
- Jira: CBRD-26744
- Components: [[components/db-value]] · [[components/base]] · [[components/cas]] · [[components/loaddb-executor]]
- Sources: [[sources/cubrid-src-base]] · [[sources/cubrid-src-compat]]
