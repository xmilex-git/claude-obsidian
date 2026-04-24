---
status: reference
type: dependency
name: "OpenSSL"
version: "1.1.1w (tarball); README lists 1.1.1f"
source: "https://github.com/CUBRID/3rdparty/raw/develop/openssl/openssl-1.1.1w.tar.gz"
license: "OpenSSL License / SSLeay License (dual; Apache-2.0-compatible for 1.1.x)"
bundled: true
used_by:
  - "TLS/SSL encrypted client-server connections"
  - "Cryptographic primitives (hashing, random)"
risk: medium
tags:
  - dependency
  - cubrid
  - security
  - tls
created: 2026-04-23
updated: 2026-04-23
---

# OpenSSL

## What it does

OpenSSL is the ubiquitous C library for TLS/SSL protocols and cryptographic primitives (symmetric ciphers, hashing, public-key operations, random number generation).

## Why CUBRID uses it

CUBRID encrypts client-server TCP connections using OpenSSL. SSL certificates are stored in `conf/` (default self-signed certs ship with the distribution). The `cubrid_log` CDC API also uses SSL connections when configured.

## Integration points

- CMake target: `libopenssl`
- Linux: built from source via `ExternalProject_Add`; produces `libssl.a` + `libcrypto.a` (static pair)
- Configure command: `<SOURCE_DIR>/config --prefix=... no-shared` (explicitly disables shared libs)
- Only `make install_sw` is run (skips man pages and other extras)
- Windows: prebuilt `libssl.lib` + `libcrypto.lib` from `win/3rdparty/openssl/`; also links `Crypt32` and `Ws2_32` system libs
- Exposes `LIBOPENSSL_LIBS` and `LIBOPENSSL_INCLUDES` to parent CMake scope
- Included in `EP_TARGETS` (Linux), `EP_LIBS`, `EP_INCLUDES` (all targets)

## Risk / notes

- **OpenSSL 1.1.1 reached end-of-life on 2023-09-11.** No further security patches from OpenSSL upstream. CUBRID pins to `1.1.1w` (final 1.1.x release, Sep 2023), which includes all 1.1.x security fixes, but future CVEs will not be patched upstream.
- Version mismatch: README says 1.1.1f (Apr 2020); actual tarball is 1.1.1w (Sep 2023). The tarball is authoritative — this is a positive drift (security patches applied).
- Migration to OpenSSL 3.x would require API changes (deprecated APIs in 3.x) — non-trivial effort.
- Medium risk: TLS termination is security-critical; EOL status is a concrete follow-up item.

> [!warning] OpenSSL 1.1.1 is EOL
> OpenSSL 1.1.1 reached end-of-life September 2023. No upstream security patches will be issued for new CVEs. Consider planning an upgrade to OpenSSL 3.x.

## Related

- [[modules/3rdparty|3rdparty module]]
- [[dependencies/_index|Dependencies]]
- [[Tech Stack]]
