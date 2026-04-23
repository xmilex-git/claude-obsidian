---
type: flow
actors: [client, broker/CAS, db_server]
entry_point: "CREATE/ALTER/DROP TABLE/INDEX/USER/VIEW; GRANT/REVOKE SQL statement"
exit_point: "schema commit + catalog updated + locks released"
triggers: [user DDL]
related_modules: [parser, object, transaction, storage]
tags:
  - flow
  - cubrid
  - ddl
  - schema
related:
  - "[[components/parser|parser]]"
  - "[[components/parse-tree|parse-tree]]"
  - "[[components/schema-manager|schema-manager]]"
  - "[[components/object|object]]"
  - "[[components/authenticate|authenticate]]"
  - "[[components/lock-manager|lock-manager]]"
  - "[[components/transaction|transaction]]"
  - "[[components/log-manager|log-manager]]"
  - "[[components/system-catalog|system-catalog]]"
  - "[[components/btree|btree]]"
  - "[[components/heap-file|heap-file]]"
  - "[[components/file-manager|file-manager]]"
  - "[[components/view-transform|view-transform]]"
  - "[[components/lob-locator|lob-locator]]"
  - "[[components/cubrid-log-cdc|cubrid-log-cdc]]"
  - "[[components/communication|communication]]"
created: 2026-04-23
updated: 2026-04-23
---

# DDL Execution Path (CREATE / ALTER / DROP / GRANT / REVOKE)

## Summary

When a CUBRID client issues a DDL statement, the SQL text is parsed entirely on the client (or CAS) into a DDL-specific `PT_NODE` (e.g. `PT_CREATE_ENTITY`, `PT_ALTER`, `PT_DROP`, `PT_GRANT`). Unlike DML, DDL never produces an XASL plan; instead the client's [[components/schema-manager|schema manager]] drives the mutation directly — acquiring a `SCH_M_LOCK` on the target class, validating privileges via `au_ctx()`, applying changes through an `SM_TEMPLATE` copy-on-write buffer, flushing catalog rows to `_db_class` / `_db_attribute` / `_db_index`, and committing within the current transaction (fully WAL-logged and rollback-safe). The server is involved only for heap/B-tree physical operations and log writes.

---

## End-to-end Sequence Diagram

```
Client / CAS                   cub_server
─────────────────────────────────────────────────────────
SQL text
  │
  ▼
[Stage 1] Parser (client-side, src/parser/)
  csql_lexer + csql_grammar.y
  → PT_CREATE_ENTITY / PT_ALTER / PT_DROP /
    PT_CREATE_INDEX / PT_GRANT / PT_REVOKE …
  name_resolution.c binds table/user names
  semantic_check.c validates structure
  NOTE: mq_translate + xasl_generation are SKIPPED
  │
  ▼
[Stage 2] Auth check (client-side, authenticate_context)
  au_ctx()->check_class_authorization(class, DB_AUTH_ALTER …)
  reads _db_auth rows; uses privilege cache
  │  DENIED → er_set → client error
  ▼
[Stage 3] Schema lock (server-side, lock_manager.c)
  NET_SERVER_TM_SERVER_LOCK_CLASSES  ──────────────────►
  lock_object(SCH_M_LOCK, class_oid)   cub_server grants
  ◄──── LK_GRANTED (or timeout/deadlock abort) ─────────
  │
  ▼
[Stage 4] Per-DDL schema mutation (client-side SM_TEMPLATE)
  smt_def_class / smt_edit_class_mop
    → mutate SM_TEMPLATE (copy-on-write)
  sm_update_class(template_, mop)
    → writes SM_CLASS descriptor
    │
    ├─ heap/btree physical ops ───────────────────────►
    │    file_create / heap_create / btree bulk-load    cub_server
    │    (LOG_UNDOREDO_DATA WAL records written)
    ◄──────────────────────────────────────────────────
  │
  ▼
[Stage 5] System catalog update (client-side, catcls_*)
  heap inserts into _db_class, _db_attribute,
  _db_index, _db_auth  (ordinary WAL-logged rows)
  │
  ▼
[Stage 6] Commit
  log_commit() → LOG_COMMIT record written
  lock_unlock_all() → SCH_M_LOCK released
  schema version bumped → remote sessions re-prepare
  statement cache invalidated
```

---

## Stage 1 — Client-side Compile

The [[components/parser|parser]] (`csql_grammar.y` + `csql_lexer.l`) converts DDL text to `PT_NODE` trees. DDL node types from [[components/parse-tree|parse-tree]]:

| SQL | `PT_NODE_TYPE` |
|-----|---------------|
| `CREATE TABLE` / `CREATE VIEW` | `PT_CREATE_ENTITY` |
| `ALTER TABLE` | `PT_ALTER` |
| `DROP TABLE` / `DROP INDEX` | `PT_DROP` |
| `CREATE INDEX` | `PT_CREATE_INDEX` |
| `CREATE USER` | `PT_CREATE_USER` (maps to DCL range) |
| `GRANT` / `REVOKE` | `PT_GRANT` / `PT_REVOKE` |
| `TRUNCATE` | `PT_TRUNCATE` |

`name_resolution.c` binds table/user names to `DB_OBJECT *` pointers; `semantic_check.c` validates structure.

> [!key-insight] DDL has no XASL
> `pt_compile()` normally ends with `xasl_generate_statement`. For DDL node types this step is **entirely skipped** — there is no XASL plan, no serialisation to the server, and no query executor involvement. The schema manager acts on the PT_NODE directly.

---

## Stage 2 — Auth Check

[[components/authenticate|authenticate]] is invoked before any mutation. `au_ctx()` returns the `authenticate_context` singleton (C++ class behind legacy `au_*` macros).

| DDL | Required privilege (`DB_AUTH`) |
|-----|-------------------------------|
| `CREATE TABLE` | `DB_AUTH_ALTER` on the schema / DBA |
| `ALTER TABLE` | `DB_AUTH_ALTER` on the class |
| `DROP TABLE` | `DB_AUTH_ALTER` on the class |
| `CREATE INDEX` | `DB_AUTH_INDEX` on the class |
| `GRANT` / `REVOKE` | `DB_AUTH_GRANT_OPTION` on the class |
| `CREATE USER` | DBA only |

The privilege cache (`authenticate_cache`) is checked first; a miss triggers a scan of `_db_auth` rows. `AU_DISABLE(save)` / `AU_ENABLE(save)` macros bypass checks for internal catalog-install operations — always paired on every exit path.

---

## Stage 3 — Schema Lock Acquisition

[[components/lock-manager|lock-manager]] uses two schema-specific modes:

| Mode | Abbrev | Who holds it |
|------|--------|-------------|
| `SCH_M_LOCK` | SCH-M | DDL — exclusive, blocks all concurrent readers/writers of that class |
| `SCH_S_LOCK` | SCH-S | DML queries — shared, prevents concurrent DDL while a query is compiling |

Lock hierarchy: Root class → Class (`SCH_M_LOCK`) → (no row locks for pure DDL).

`lock_object(thread_p, class_oid, root_class_oid, SCH_M_LOCK, LK_UNCOND_LOCK)` blocks until all active DML against the class completes. Timeout is governed by `PRM_ID_LK_TIMEOUT_SECS`; expiry returns `LK_NOTGRANTED_DUE_TIMEOUT`.

> [!key-insight] SCH-M blocks queries globally
> Any session that has compiled but not yet executed a query against the same table holds `SCH_S_LOCK`. DDL must wait for all of them to release before the `SCH_M_LOCK` is granted — even read-only `SELECT` statements.

---

## Stage 4 — Per-DDL Specifics

All mutations use the [[components/schema-manager|schema manager]] template pattern: open `SM_TEMPLATE`, accumulate changes, call `sm_update_class()`.

### CREATE TABLE

```
smt_def_class("t")          → new SM_TEMPLATE
smt_add_attribute_w_dflt()  → add column definitions
sm_update_class(tmpl, mop)
  → file_create(FILE_HEAP) via [[components/file-manager]]
  → heap_create() empty HFID via [[components/heap-file]]
  → catalog row inserted into _db_class, _db_attribute
```

If the table has LOB (`BLOB`/`CLOB`) columns, `lob_locator_add()` is called per LOB value later at DML time — DDL itself only records the domain in `_db_attribute`. See [[components/lob-locator|lob-locator]].

### CREATE INDEX

```
sm_add_constraint(classop, DB_CONSTRAINT_INDEX, …)
  → btree_load (btree_load.c): sort existing heap rows,
    bulk-fill leaf pages, build internal nodes bottom-up
  → file_create(FILE_INDEX) via [[components/file-manager]]
  → _db_index + _db_index_key catalog rows inserted
```

[[components/btree|btree]] `btree_load.c` implements the bulk-load path (sort + sequential leaf fill), which is significantly faster than per-row insert. `SCH_M_LOCK` is held for the full duration unless the online index build path (`btree_online_index_dispatcher`) is used.

> [!key-insight] Online vs offline index build
> Standard `CREATE INDEX` holds `SCH_M_LOCK` and blocks all DML for its duration. The online index dispatcher (`btree_online_index_list_dispatcher`) captures concurrent inserts/deletes during the bulk-load phase and replays them afterward, allowing DML to continue.

### ALTER TABLE

Column add/drop/modify goes through `smt_edit_class_mop` → mutations → `sm_update_class`. Adding a NOT NULL column with no default requires a heap scan to validate existing rows. Changing a column's type that backs an index triggers: drop old index (`sm_drop_index`), alter column domain, recreate index via bulk-load. Large restructuring may invoke `heap_compact` (online).

### DROP TABLE

```
sm_delete_class_mop(op, is_cascade_constraints)
  → catalog rows in _db_class, _db_attribute, _db_index deleted immediately
  → heap file destruction logged as LOG_POSTPONE (deferred redo)
  → index B-tree files similarly deferred
```

> [!key-insight] File destruction is deferred (LOG_POSTPONE)
> Physical `file_destroy()` for the heap and index files is not executed at `sm_delete_class_mop` time. It is registered as a `LOG_POSTPONE` operation and run by `log_do_postpone()` inside `log_commit()`. This means a rollback of `DROP TABLE` cleanly restores the table — the file was never actually destroyed until the transaction committed.

### CREATE VIEW

```
smt_def_class("v")  with SM_VCLASS_CT type
  → view query string stored in _db_class.comment / class descriptor
  → no heap, no index created
  → mq_translate (view_transform.c) is invoked at query time, not here
```

The [[components/view-transform|view-transform]] module (`mq_translate`) is **not** called during `CREATE VIEW`. It runs at query compile time to inline the view definition. `WITH CHECK OPTION` flag is stored in `SM_CLASS_HEADER` as `SM_CLASSFLAG_WITHCHECKOPTION`.

### CREATE USER / GRANT / REVOKE

```
CREATE USER  → insert into db_user (_db_user) + db_password
GRANT        → insert/update row in _db_auth (grantee, class, privileges, grant_option)
REVOKE       → delete/update row in _db_auth
             → au_reset_authorization_caches() flushes privilege cache
```

Backed entirely by [[components/authenticate|authenticate]] and [[components/system-catalog|system-catalog]] (`CT_CLASSAUTH_NAME` = `_db_auth`). `AU_DISABLE` is used internally so the catalog writes bypass the very privilege system being mutated.

### CREATE TRIGGER

Trigger DDL writes to `_db_trigger` (`CT_TRIGGER_NAME`) via `trigger_manager.c` (`tr_create_trigger`). The trigger manager is not yet a dedicated wiki component page — see `src/object/trigger_manager.c`.

---

## Stage 5 — System Catalog Update

[[components/system-catalog|system-catalog]] physical tables (`_db_class`, `_db_attribute`, `_db_index`, `_db_index_key`, `_db_auth`, etc.) are updated as **ordinary heap inserts/updates/deletes on system tables**. These go through the same `heap_insert_logical` / WAL path as user data — they are not special-cased at the storage layer.

`catcls_*` functions (in `schema_system_catalog_install.cpp`) provide the wrappers that translate `SM_CLASS` / `SM_ATTRIBUTE` structs into `DB_VALUE` columns and execute the writes.

---

## Stage 6 — Commit

```
log_commit(thread_p, tran_index, retain_lock=false)
  ├── log_do_postpone()    ← runs deferred file_destroy for DROP TABLE
  ├── log_complete()       ← writes LOG_COMMIT record to WAL
  ├── lock_unlock_all()    ← releases SCH_M_LOCK (DML can resume)
  └── logtb_free_tran_index()
```

Schema version is bumped in the catalog. Remote CAS sessions that have a cached prepared statement for the affected table will detect the version mismatch at next reference and transparently re-prepare.

> [!key-insight] Schema version invalidates remote prepared statements
> CUBRID tracks a per-class schema version in `_db_class`. When a remote session next executes a prepared statement, the CAS compares the cached schema version with the current catalog value. On mismatch it silently discards the cached plan and re-prepares — there is no explicit broadcast; invalidation is lazy and pull-based.

### CDC / Supplemental Logging

If `supplemental_log` is enabled on the class (`SM_CLASSFLAG_SUPPLEMENTAL_LOG`), `LOG_SUPPLEMENTAL_INFO` (type 52) records are appended to the WAL. These carry before/after schema images for the [[components/cubrid-log-cdc|cubrid-log-cdc]] Change Data Capture API. This is additive to ARIES logging and does not affect recovery correctness.

---

## Failure Modes

| Failure | When | Effect |
|---------|------|--------|
| Auth denied | Stage 2 | `er_set(ER_ERROR_SEVERITY, …)` returned; no mutation |
| `SCH_M_LOCK` timeout | Stage 3 | `LK_NOTGRANTED_DUE_TIMEOUT`; DDL aborted, no mutation |
| Deadlock during lock wait | Stage 3 | Youngest txn chosen as victim; DDL aborted |
| Constraint violation during ALTER | Stage 4 | Heap scan finds violating row; `sm_update_class` fails; template discarded via `smt_quit` |
| OOM in SM_TEMPLATE | Stage 4 | `longjmp` via `PT_SET_JMP_ENV`; parser arena freed |
| Partial DDL rollback | Any | DDL is fully transactional; `log_abort()` undoes all catalog rows and queues `file_destroy` for any newly created heap/index files |

---

## Notable Contrasts with DML

| Aspect | DDL | DML |
|--------|-----|-----|
| Plan type | None — schema manager acts on PT_NODE directly | XASL plan compiled + serialised to server |
| Lock mode | `SCH_M_LOCK` on class | `IS_LOCK`/`IX_LOCK` on class, `S_LOCK`/`X_LOCK` on rows |
| Catalog mutation | Writes `_db_class`, `_db_attribute`, etc. | Writes user table heap pages |
| Prepared statement impact | Invalidates all remote prepared stmts for affected table | None |
| File lifecycle | `file_create` / `file_destroy` (deferred) | Never touches file allocation |
| Execution side | Client-side schema manager (CS_MODE) or in-process (SA_MODE) | Server-side XASL executor |
| MVCC involvement | Catalog rows use MVCC but DDL itself holds `SCH_M_LOCK` | Full MVCC snapshot per row |

---

## Cross-References

- [[components/parser|parser]] — SQL text to `PT_NODE`; DDL node types listed in [[components/parse-tree|parse-tree]]
- [[components/schema-manager|schema-manager]] — `sm_update_class`, `sm_delete_class_mop`, template pattern
- [[components/object|object]] — parent hub: schema, auth, catalog, LOB, workspace
- [[components/authenticate|authenticate]] — `au_ctx()`, `DB_AUTH` privilege bits, `AU_DISABLE` pattern
- [[components/lock-manager|lock-manager]] — `SCH_M_LOCK` / `SCH_S_LOCK` acquire/release
- [[components/transaction|transaction]] — DDL is fully transactional; ARIES WAL; `log_commit`
- [[components/log-manager|log-manager]] — `LOG_POSTPONE` for deferred file destroy; `LOG_SUPPLEMENTAL_INFO` for CDC
- [[components/system-catalog|system-catalog]] — `_db_class`, `_db_attribute`, `_db_index`, `_db_auth` physical tables
- [[components/btree|btree]] — `btree_load.c` bulk-load path for `CREATE INDEX`
- [[components/heap-file|heap-file]] — heap creation/destruction for `CREATE`/`DROP TABLE`
- [[components/file-manager|file-manager]] — `file_create` / `file_destroy` (deferred via `LOG_POSTPONE`)
- [[components/view-transform|view-transform]] — `mq_translate` runs at query time, not at `CREATE VIEW`
- [[components/lob-locator|lob-locator]] — LOB columns have cross-cutting state machine effects at DML time
- [[components/cubrid-log-cdc|cubrid-log-cdc]] — `LOG_SUPPLEMENTAL_INFO` gates DDL CDC events
- [[components/communication|communication]] — DDL lock and physical ops use same `NET_SERVER_*` dispatch infrastructure
- [[flows/_index|Flows index]]
