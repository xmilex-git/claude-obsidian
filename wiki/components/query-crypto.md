---
type: component
parent_module: "[[modules/src|src]]"
path: "src/query/crypt_opfunc.c"
status: active
purpose: "Cryptographic built-in functions: MD5, SHA1/2, AES-128-ECB, DES-ECB, CRC32, random bytes, hex encoding, and DBLink transport-layer crypto"
key_files:
  - "crypt_opfunc.c"
  - "crypt_opfunc.h"
tags:
  - component
  - cubrid
  - query
  - crypto
related:
  - "[[components/query-string|query-string]]"
  - "[[components/dblink|dblink]]"
  - "[[dependencies/openssl|openssl]]"
  - "[[components/db-value|db-value]]"
  - "[[Memory Management Conventions]]"
created: 2026-04-23
updated: 2026-04-23
---

# `crypt_opfunc.c` — Cryptographic Built-ins

Server-side implementations of MD5, SHA-1, SHA-2 family, AES-128-ECB, DES-ECB block ciphers, CRC32, hex encoding, random-byte generation, and a suite of DBLink-specific transport-layer crypto helpers. All SQL-callable functions delegate here from [[components/query-string|query-string]].

## Purpose

`crypt_opfunc.c` is the single point of contact between CUBRID's SQL layer and OpenSSL's EVP cryptography API (`<openssl/evp.h>`, `<openssl/sha.h>`, `<openssl/rand.h>`). It also contains a non-OpenSSL CRC32 implementation (via `CRC.h`) and a DBLink-specific AES-based key-exchange helper.

## Public Entry Points

| Signature | Role |
|-----------|------|
| `crypt_default_encrypt(thread_p, src, src_len, key, key_len, dest_p, dest_len_p, enc_type)` | AES_128_ECB or DES_ECB block encryption; allocates `dest_p` via `db_private_alloc` |
| `crypt_default_decrypt(thread_p, src, src_len, key, key_len, dest_p, dest_len_p, enc_type)` | Decrypt counterpart; same allocation pattern |
| `crypt_sha_one(thread_p, src, src_len, dest_p, dest_len_p)` | SHA-1 → raw binary (20 bytes); caller converts to hex |
| `crypt_sha_two(thread_p, src, src_len, need_hash_len, dest_p, dest_len_p)` | SHA-2 family: `need_hash_len` ∈ {224, 256, 384, 512}; raw binary output |
| `crypt_md5_buffer_hex(buffer, len, resblock)` | MD5 → 32-char lowercase hex + NUL; writes into caller-supplied `resblock[33]` |
| `str_to_hex(thread_p, src, src_len, dest_p, dest_len_p, lettercase)` | Binary → hex string; allocates via `db_private_alloc` |
| `str_to_hex_prealloced(src, src_len, dest, dest_len, lettercase)` | Same but writes into caller-supplied buffer |
| `crypt_generate_random_bytes(dest, length)` | CSPRNG via `RAND_bytes(OpenSSL)` |
| `crypt_crc32(src, src_len, dest)` | CRC32 → `int*` (not OpenSSL; uses bundled `CRC.h`) |
| `crypt_dblink_encrypt / crypt_dblink_decrypt` | DBLink AES transport layer (see below) |
| `shake_dblink_password / reverse_shake_dblink_password` | DBLink password obfuscation |
| `crypt_dblink_bin_to_str / crypt_dblink_str_to_bin` | DBLink binary ↔ base64 encoding with XOR key |

## SQL → C Delegation Path

SQL functions in [[components/query-string|query-string]] delegate here:

```
db_string_md5(val, result)          → crypt_md5_buffer_hex()
db_string_sha_one(val, result)      → crypt_sha_one() → str_to_hex()
db_string_sha_two(src, len, result) → crypt_sha_two() → str_to_hex()
db_string_aes_encrypt(src, key, result) → crypt_default_encrypt(..., AES_128_ECB)
db_string_aes_decrypt(src, key, result) → crypt_default_decrypt(..., AES_128_ECB)
db_crc32_dbval(result, value)       → crypt_crc32()
```

## Execution Path

```
fetch_func_value (fetch.c) or query_opfunc.c
  → db_string_md5(val, result) [string_opfunc.c]
      → crypt_md5_buffer_hex(db_get_string(val), len, hex_buf)
          → crypt_md5_buffer_binary(buf, len, binary_md5)  // OpenSSL EVP_DigestInit_ex / EVP_DigestFinal_ex
          → str_to_hex_prealloced(binary_md5, 16, hex_buf, 33, HEX_LOWERCASE)
      → db_make_varchar(result, 32, hex_buf, ...)
```

## OpenSSL API Usage

> [!key-insight] Uses EVP (high-level OpenSSL API) exclusively
> `crypt_opfunc.c` uses the `EVP_*` family, not the deprecated direct SHA/AES interfaces. This makes it forward-compatible with OpenSSL 3.x which deprecates `SHA1()`, `SHA256()` etc. in favour of `EVP_DigestInit_ex`. The EVP context is created, used, and destroyed within each call — no long-lived OpenSSL state.

The AES key generation (`aes_default_gen_key`) mimics MySQL's 128-bit AES key derivation: XOR-fold the user key bytes into a 16-byte array. This is **not** a KDF; it is a compatibility behaviour with MySQL's `AES_ENCRYPT`.

## Input / Output Formats

| Function | Input | Output format |
|---------|-------|---------------|
| `MD5(str)` | VARCHAR | VARCHAR(32) lowercase hex |
| `SHA1(str)` | VARCHAR | VARCHAR(40) lowercase hex |
| `SHA2(str, len)` | VARCHAR, INT | VARCHAR(N) lowercase hex where N ∈ {56,64,96,128} |
| `AES_ENCRYPT(str, key)` | VARCHAR, VARCHAR | VARBIT (raw binary ciphertext, padded to block boundary) |
| `AES_DECRYPT(str, key)` | VARBIT, VARCHAR | VARCHAR (decrypted plaintext) |
| `CRC32(str)` | VARCHAR | INTEGER (signed 32-bit) |
| `GUID()` | — | VARCHAR(36) lowercase UUID-format (from random bytes, formatted with hyphens in [[components/query-string\|query-string]]) |

## DBLink Crypto Sub-system

`crypt_opfunc.c` also implements a separate key-exchange and message-authentication scheme for [[components/dblink|dblink]] transport:

- `crypt_dblink_encrypt / crypt_dblink_decrypt` — AES-128-ECB using a master key `mk` of `TDE_DATA_KEY_LENGTH` bytes (aliased via `DBLINK_CRYPT_KEY_LENGTH`).
- `shake_dblink_password` — XOR-obfuscates a plaintext password using the current timestamp as entropy (not cryptographically strong; designed for transit obfuscation only).
- `crypt_dblink_bin_to_str / crypt_dblink_str_to_bin` — custom base64-like encoding with a XOR key and time component.

> [!warning] DBLink password crypto is weak by design
> `shake_dblink_password` is time-seeded XOR, not a real cipher. It protects against passive sniffing on local IPC but not against a determined adversary with access to the communication channel. DBLink traffic should be secured at the network layer (TLS) for production environments.

## Constraints

### NULL Handling
All SQL-facing wrappers in `string_opfunc.c` check for NULL before calling here. `crypt_*` functions themselves do not check for NULL inputs; they expect valid pointers.

### Memory Ownership
Functions that produce variable-length output (`crypt_default_encrypt`, `crypt_sha_one`, `crypt_sha_two`, `str_to_hex`) allocate via `db_private_alloc(thread_p, ...)`. The `dest_p` output pointer is owned by the caller's private heap and must be freed with `db_private_free_and_init`. In SERVER_MODE, if `thread_p == NULL`, `thread_get_thread_entry_info()` is called to recover it.

### Threading
OpenSSL EVP functions are thread-safe as of OpenSSL 1.1+ (no global state). `RAND_bytes` is thread-safe. `crypt_crc32` (non-OpenSSL) operates on call-stack data only and is thread-safe.

### Error Model
Returns `NO_ERROR` (0) on success or a negative CUBRID error code. OpenSSL internal errors are mapped to `ER_CSS_OPENSSL_CRYPT_*` codes. Allocation failures return `ER_OUT_OF_VIRTUAL_MEMORY`.

## Lifecycle

- No per-server initialization required (OpenSSL auto-init since 1.1).
- All functions are stateless beyond their arguments.
- Called per-tuple from `string_opfunc.c` wrappers.

## Related

- [[components/query-string|query-string]] — SQL-facing wrappers (MD5, SHA*, AES_ENCRYPT, AES_DECRYPT, CRC32, GUID)
- [[components/dblink|dblink]] — DBLink crypto sub-system consumer
- [[dependencies/openssl|openssl]] — upstream EVP crypto library
- [[Memory Management Conventions]]
