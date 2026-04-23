---
type: component
parent_module: "[[components/object|object]]"
path: "src/object/"
status: active
purpose: "User authentication, group membership, privilege caching, password management, execution-rights stack for stored procedures"
key_files:
  - "authenticate.c / authenticate.h (au_ macro aliases + legacy interface)"
  - "authenticate_context.hpp / authenticate_context.cpp (authenticate_context C++ class — the real implementation)"
  - "authenticate_cache.hpp / authenticate_cache.cpp (per-class privilege cache)"
  - "authenticate_password.hpp / authenticate_password.cpp (password hashing: DES-old, SHA-1, SHA2-512)"
  - "authenticate_constants.h (DB_AUTH privilege bit flags)"
public_api:
  - "au_ctx() — returns authenticate_context* singleton (thread-local)"
  - "AU_DISABLE(save) / AU_ENABLE(save) — bypass privilege checks (macros)"
  - "AU_SAVE_AND_ENABLE(save) — temporarily enable while saving old state"
  - "au_check_user() — assert current user is logged in"
  - "au_set_user(newuser) — change current user (alias for au_ctx()->set_user)"
  - "au_login(name, password, ignore_dba_privilege)"
  - "au_install() — bootstrap auth tables at DB creation"
  - "au_start() — load auth state at DB open"
  - "au_perform_push_user(user) / au_perform_pop_user() — SP execution rights stack"
  - "AU_DISABLE_PASSWORDS() — skip password validation (utility programs)"
tags:
  - component
  - cubrid
  - auth
  - security
  - client
related:
  - "[[components/object|object]]"
  - "[[components/system-catalog|system-catalog]]"
  - "[[components/schema-manager|schema-manager]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-04-23
---

# Authorization Manager (`src/object/authenticate*.c/h/hpp`)

The authorization manager controls which users can access which database objects. It is client-side only (no `SERVER_MODE`).

> [!warning] Client-side only
> `authenticate.h` contains `#error Does not belong to server module`. All privilege checks execute on the client. The server trusts the client library to enforce access control before sending requests.

## Architecture: `authenticate_context`

The implementation was refactored from global variables into a C++ class `authenticate_context` (in `authenticate_context.hpp`). The legacy `au_*` macros in `authenticate.h` are thin aliases:

```cpp
#define Au_root       au_ctx()->root
#define Au_user       au_ctx()->current_user
#define Au_dba_user   au_ctx()->dba_user
#define Au_public_user au_ctx()->public_user
#define Au_disable    au_ctx()->disable_auth_check

#define au_init       au_ctx          // same call
#define au_install    au_ctx()->install
#define au_start      au_ctx()->start
#define au_set_user   au_ctx()->set_user
```

The context holds:
- `root` — MOP of the `db_root` object (authorization root)
- `current_user` — MOP of the current logged-in user (`_db_user` instance)
- `dba_user`, `public_user` — cached MOPs for system users
- `caches` (`authenticate_cache`) — per-class privilege bitmask cache
- `user_stack` (`std::stack<MOP>`) — execution-rights stack for stored procedures
- Password buffers (SHA2-512, SHA-1, DES-old-style, all stored separately)
- `disable_auth_check` flag

## User Model

Users are stored in `_db_user` (`CT_USER_NAME` / `AU_USER_CLASS_NAME`). Every user is automatically a member of `PUBLIC`. The `DBA` user is implicitly a member of every group.

| User/Group | Behavior |
|------------|----------|
| `PUBLIC` | All users are implicit members; grants to PUBLIC are visible to all |
| `DBA` | Implicit member of all groups; all permissions |
| Named user | Explicit member of zero or more groups |

Authorization grants are stored in `_db_auth` (`CT_CLASSAUTH_NAME`). Each row records `(grantee, class, privileges, grant_option)`.

## Privilege Bits (`authenticate_constants.h`)

Privileges are bitmasks of type `DB_AUTH`:
- `DB_AUTH_SELECT`, `DB_AUTH_INSERT`, `DB_AUTH_UPDATE`, `DB_AUTH_DELETE`
- `DB_AUTH_ALTER`, `DB_AUTH_INDEX`, `DB_AUTH_EXECUTE`
- Combined: `DB_AUTH_ALL`

## Lifecycle

| Phase | Call | What happens |
|-------|------|--------------|
| DB creation | `au_install()` | Create `db_root`, `db_user`, `db_password`, `db_authorization` system classes; create DBA and PUBLIC users |
| DB open | `au_start()` | Load `db_root`, cache `Au_root`, set current user based on registered name |
| Login | `au_login(name, pw, ignore_dba)` | Validate password against stored hashes; set `current_user` MOP |
| Privilege check | `au_check_class_authorization(class, auth)` | Look up per-class cache; if miss, scan `_db_auth`; cache result |
| Disable for internal | `AU_DISABLE(save)` | Set `disable_auth_check = true` — used in bootstrap and internal DDL |
| SP execution | `au_perform_push_user(user)` | Push new user context for `DEFINER` rights during SP execution |

> [!key-insight] Privilege cache invalidation
> `authenticate_cache` caches privilege bitmasks per class per user. The cache is invalidated when `GRANT`/`REVOKE` is executed or when the user changes. A stale cache can cause apparent permission errors or over-grants between DDL statements — clear with `au_reset_authorization_caches()`.

## Password Hashing

Three hash formats are stored per user (`_db_user.password`):
- DES old-style (legacy compatibility)
- SHA-1 (deprecated but retained)
- SHA2-512 (current default)

`AU_DISABLE_PASSWORDS()` skips password validation entirely — used by `cub_server` utility programs that rely on OS-level file permissions instead.

## Execution Rights Stack (Stored Procedures)

`authenticate_context.user_stack` is a `std::stack<MOP>`. When a stored procedure with `DEFINER` rights executes:
1. `au_perform_push_user(definer_mop)` — temporarily switch current user.
2. SP body runs with definer's privileges.
3. `au_perform_pop_user()` — restore caller's user.

## `AU_DISABLE` / `AU_ENABLE` Pattern

Used extensively throughout the schema manager and catalog install code to bypass privilege checks for internal operations:

```c
int save;
AU_DISABLE (save);          // save = old flag, set disable_auth_check = true
/* ... internal operations that must not fail on privilege check ... */
AU_ENABLE (save);           // restore original flag
```

> [!warning] Leaving AU_DISABLE active
> If an error path returns without calling `AU_ENABLE`, subsequent user SQL runs without authorization. Always pair these macros with an error-exit path that restores the flag.

## Auth-Related Catalog Tables

| Table | Role |
|-------|------|
| `db_root` | Authorization root object (singleton) |
| `db_user` (`_db_user`) | User records |
| `db_password` | Hashed passwords |
| `db_authorization` | Per-user authorization objects |
| `_db_auth` | Grant rows: grantee × class × privilege bits |

## Related

- Parent: [[components/object|object]]
- [[components/system-catalog|system-catalog]] — `AUTH_CHECK_CLASS()` macro used in info-schema view specs
- [[components/schema-manager|schema-manager]] — `sm_update_class_with_auth` calls au_ before commit
- [[Build Modes (SERVER SA CS)]] — client-only
