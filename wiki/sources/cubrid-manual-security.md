---
created: 2026-04-27
type: source
title: "CUBRID Manual — Security (security.rst)"
source_path: "/home/cubrid/cubrid-manual/en/security.rst"
ingested: 2026-04-27
status: complete
tags:
  - source
  - cubrid
  - manual
  - security
  - tde
  - ssl
  - acl
  - authorization
related:
  - "[[sources/cubrid-manual-en-overview]]"
  - "[[sources/cubrid-manual-sql-foundation]]"
  - "[[components/authenticate]]"
  - "[[components/double-write-buffer]]"
  - "[[components/log-manager]]"
---

# CUBRID Manual — Security (security.rst)

**Ingested:** 2026-04-27
**Source file:** `/home/cubrid/cubrid-manual/en/security.rst` (375 lines)

## What This Covers

The four pillars of CUBRID security:
1. **Packet Encryption** — SSL/TLS for client ↔ server traffic
2. **ACL** (Access Control List) — IP-based access at broker + server layers
3. **Authorization** — user/group privileges (cross-references to `sql/authorization.rst`)
4. **TDE** (Transparent Data Encryption) — at-rest encryption with two-level keys

## Section Map

| Section | Content |
|---|---|
| **Packet Encryption** | SSL/TLS fundamentals, MITM threat, OpenSSL backend, supported protocols (SSLv3, TLSv1, TLSv1.1, TLSv1.2), self-signed cert + private key in `$CUBRID/conf/cas_ssl_cert.{crt,key}`, OpenSSL commands to replace |
| **ACL (Access Control List)** | Two-layer ACL — broker layer + server layer; cross-refs to admin/control.rst |
| **Authorization** | Cross-ref to `sql/authorization.rst`; user/group + GRANT/REVOKE |
| **TDE Concept** | Engine-level encryption; transparent to application; ENCRYPT clause on CREATE TABLE |
| **TDE Key Management** | Two-level (master + data); master in `<db>_keys` file (up to 128); managed by `cubrid tde` utility |
| **TDE File-based Key Management** | `<db>_keys` location, `tde_keys_file_path` parameter, key add/delete/change/list semantics |
| **TDE Encryption Target** | What's encrypted: permanent (table+index data), temporary (sort/spill files), log (REDO/UNDO), DWB, backup |
| **TDE on HA** | Per-node independent TDE config; replicated table requires both nodes have the same keys |
| **TDE Restrictions** | Replication log NOT encrypted; cannot ALTER TABLE to add/remove TDE; SQL log not encrypted |
| **TDE Algorithm** | AES-256 (default) or ARIA-256; `tde_default_algorithm` parameter |
| **TDE Check** | `SHOW CREATE TABLE`, `SELECT tde_algorithm FROM db_class`, `cubrid diagdb -d1` |

## Key Facts

### SSL/TLS Packet Encryption
- **Default = OFF**. Enable per-broker with `SSL = ON` in `cubrid_broker.conf`. Restart broker to apply.
- **OpenSSL** is the implementation library on the server side.
- **Drivers that support SSL**: JDBC, CCI. Other drivers (PHP, ODBC, etc.) — explicitly not enumerated as supported.
- Default cert/key: `$CUBRID/conf/cas_ssl_cert.crt` and `cas_ssl_cert.key` — self-signed, replaceable with CA-issued or freshly generated:
  ```
  openssl genrsa -out my_cert.key 2048
  openssl req -new -key my_cert.key -out my_cert.csr
  openssl x509 -req -days 365 -in my_cert.csr -signkey my_cert.key -out my_cert.crt
  ```
- Client opts in via `useSSL=true` URL parameter (JDBC + CCI).
- **Mismatch = connection rejection at broker** (not negotiation): if broker `SSL=ON` and driver `useSSL=false`, broker rejects.

### ACL
- **Two layers** — broker layer (`ACCESS_CONTROL` + `ACCESS_CONTROL_FILE`) and server layer (`access_ip_control` + `access_ip_control_file`).
- Broker ACL controls WAS/web tier access. Server ACL controls broker/csql access.
- **11.4 new**: `ACCESS_CONTROL_DEFAULT_POLICY = DENY | ALLOW` — what to do when a broker is not listed in the ACL file. Default `DENY`.
- `cubrid broker acl status/reload` — operational verbs.
- ACL file syntax: `[%<broker_name>]` sections, lines `<dbname>:<user>:<acl_ip_list_file>`.

### TDE Concept
- **Encryption granularity = table**. `CREATE TABLE … ENCRYPT=AES;` (or `ENCRYPT=ARIA`).
- **Cannot ALTER an existing table to add/remove TDE.** Must export, drop, recreate, reimport.
- All disk reads/writes for the encrypted table are auto-encrypted/decrypted.

### Two-level keys
- **Master key** — in `<db>_keys` file. **Up to 128 keys** in the file. **Master key encrypts data keys**. DBA-managed via `cubrid tde` utility.
- **Data key** — actual symmetric key used to encrypt table/log/temp/DWB data. Lives in DB volume header, **always stored encrypted under the current master key**. Engine-managed.
- **Two-level scheme advantage**: rekey is fast — re-encrypt only the (small) data keys, not all data.
- **Loss of master key = data unrecoverable.** Manual is explicit.

### Key file lifecycle
- Default `<db>_keys` location = same directory as data volume.
- Override via `tde_keys_file_path` system parameter.
- Created automatically by `cubrid createdb` (one initial master key).
- DBA can add/delete/change/list keys via `cubrid tde -a/-d/-c/-s/-n` etc.
- Cannot delete the currently-active master key. Can change it (must have both old and new in the file).
- **Key inquiry** (`cubrid tde --show-keys <db>`) shows: current key index, creation/set timestamps, key count.

### What gets encrypted
- **Permanent**: table data + all indexes on that table.
- **Temporary**: sort/spill files generated for queries on TDE tables (e.g., `ORDER BY`).
- **Log (REDO/UNDO)**: active and archive logs for changes to TDE tables.
- **DWB (Double Write Buffer)**: pages destined for TDE tables.
- **Backup**: TDE data stays encrypted in backup volumes.

### What does NOT get encrypted
- Replication log (HA copylogdb stream) — explicit gap.
- SQL log (broker-side logging).

### Backup with TDE
- Backup volume **includes the key file by default** (security risk — backup leak = master key leak).
- **`--separate-keys`** flag exports key file separately as `<db>_bk<level>_keys`. Backup volume becomes useless without the separate key file.
- Restore key-file lookup priority: (1) backup volume's embedded key file, (2) separate `--separate-keys` file, (3) `tde_keys_file_path` server key file, (4) default location server key file.
- **Incremental restore**: uses `--level` key file; if not specified, uses highest-level key file.

### TDE algorithms
- **AES-256** — default, hardware-accelerated, NIST standard.
- **ARIA-256** — Korean national standard, lightweight HW.
- Per-table override via `ENCRYPT=AES|ARIA`.
- Default for non-table data (logs, temp, DWB): `tde_default_algorithm` parameter (default AES).

### TDE in HA
- TDE applies **per node, independently**. Each node has its own `<db>_keys` file.
- For replicated TDE tables, both master and slave must have valid TDE setup. Otherwise replication stops on first TDE-table change.
- Post-restart with valid TDE config, replication resumes from where it stopped.

### Checking encryption
- `SHOW CREATE TABLE <name>` → shows `ENCRYPT=AES|ARIA`.
- `SELECT class_name, tde_algorithm FROM db_class WHERE class_name LIKE '...'` → `'AES' | 'ARIA' | 'NONE'`.
- `cubrid diagdb -d1 <db>` → file header includes `tde_algorithm:` line.

## Cross-References

- [[components/authenticate]] — user/group/GRANT/REVOKE implementation
- [[sources/cubrid-manual-sql-foundation]] (`sql/authorization.rst`) — SQL-level auth syntax
- [[components/double-write-buffer]] — DWB encryption
- [[components/log-manager]] — log encryption
- [[sources/cubrid-manual-admin]] — `cubrid tde`, `cubrid backupdb --separate-keys`, `cubrid restoredb --keys-file-path`

## Incidental Wiki Enhancements

- [[components/authenticate]]: documented two-layer ACL model (broker + server) and `ACCESS_CONTROL_DEFAULT_POLICY = DENY|ALLOW` (new in 11.4).
- [[components/double-write-buffer]]: documented that DWB pages destined for TDE tables are themselves encrypted in DWB.
- [[components/log-manager]]: documented that REDO/UNDO log records for TDE-table changes are encrypted both in active and archive log volumes; replication log is NOT encrypted (explicit gap).

## Key Insight

CUBRID's security story has three sharp edges:
1. **TDE rekey is cheap** (only the small key file gets re-encrypted), but **TDE cannot be added or removed from existing tables** — table re-creation required.
2. **HA does NOT encrypt replication log** — security boundary stops at the wire between master and slave.
3. **Master key loss = data loss.** Backup of the `<db>_keys` file is mandatory and should be **separate** from the data backup (use `--separate-keys`).
