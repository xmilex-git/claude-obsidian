---
type: component
parent_module: "[[modules/contrib|contrib]]"
path: "contrib/python/, contrib/python-obsolete/, contrib/php5/, contrib/php4/, contrib/perl/, contrib/ruby/, contrib/adodotnet/, contrib/hibernate/"
status: developing
purpose: "Contributor-maintained language drivers — each is a thin binding over CCI (C) or JDBC; not part of the engine CMake build"
tags:
  - component
  - cubrid
  - drivers
  - contrib
  - cci
  - jdbc
related:
  - "[[modules/contrib|contrib]]"
  - "[[modules/cubrid-cci|cubrid-cci]]"
  - "[[modules/cubrid-jdbc|cubrid-jdbc]]"
  - "[[sources/cubrid-contrib|cubrid-contrib]]"
created: 2026-04-23
updated: 2026-04-23
---

# Contrib Language Drivers

The `contrib/` tree contains contributor-maintained bindings for eight language/framework targets. All of them are thin wrappers: they convert language-native call conventions to either the [[modules/cubrid-cci|CCI C library]] (for C-extension drivers) or [[modules/cubrid-jdbc|JDBC]] (for JVM-based frameworks). None speaks the CSS wire protocol directly.

## Python — `contrib/python/`

| Item | Detail |
|------|--------|
| Package name | `CUBRIDdb` |
| Standard | Python DB API 2.0 (PEP 249) |
| Authors | Li Jinhu, Li Lin, Zhang Hui (NHN, 2012) |
| Extras | `Django_cubrid` — Django ORM backend for CUBRID |
| Dependency | CUBRID 8.4.0+; Python 2.4+ or 3.0+ |
| Build | `python setup.py build && python setup.py install` |
| License | (see project; APIs historically BSD) |

The C extension module `_cubrid` wraps CCI at the C level; `cubriddb.py` is the pure-Python DB API façade. A `samples/` directory ships with usage examples.

## Python (Obsolete) — `contrib/python-obsolete/`

Earlier Python 2-only driver by Kang Dong-Wan (NHN). BSD licensed. Requires CUBRID 2008 R1.1+. Superseded by `contrib/python/` which added Python 3 support and Django backend. Retained for reference only.

## PHP5 — `contrib/php5/`

Official PHP extension for CUBRID. Built as a PECL-style C extension wrapping CCI. Distributed under BSD license. Upstream docs at `http://www.cubrid.org/wiki_apis/entry/cubrid-php-driver`. Supports PHP 5.x.

## PHP4 — `contrib/php4/`

Legacy PHP4 driver. Archived; superseded by `contrib/php5/`. Retained for historical completeness.

## Perl — `contrib/perl/`

`DBD::cubrid` — a DBI-compliant Perl database driver for CUBRID. Author: Zhang Hui. Build via standard DBI/DBD Perl toolchain:

```
perl Makefile.PL
make && make test && make install
```

Windows install via ActivePerl PPM (`ppm install DBD::cubrid`). Linux via CPAN (`install DBD::cubrid`).

## Ruby — `contrib/ruby/`

Ruby adapter for CUBRID. No README was found at inventory time; directory exists in the tree. Likely wraps CCI via a C extension following the standard Ruby `mysql2`-style pattern.

## ADO.NET — `contrib/adodotnet/`

ADO.NET data provider for .NET clients. Allows C# / VB.NET applications to connect to CUBRID through the standard `System.Data` provider model. Wraps the CUBRID JDBC or CCI layer.

## Hibernate — `contrib/hibernate/`

Hibernate ORM dialect for CUBRID. JVM-based; depends on [[modules/cubrid-jdbc|cubrid-jdbc]] (the JDBC submodule). Provides:
- `CUBRIDDialect` — Hibernate SQL dialect mapping CUBRID types, auto-increment, sequence idioms
- Connection via standard JDBC URL (`jdbc:cubrid:…`)

No separate README was found in the directory at inventory time; the dialect class itself is the primary artifact.

## Common Pattern

Every language driver follows the same layered structure:

```
Language runtime
    └─ Driver adapter (language-native, e.g. Python C-ext / PHP C-ext / Ruby C-ext)
           └─ CCI C library  (cubrid-cci submodule)
                  └─ CUBRID broker / CAS process
                         └─ cub_server
```

Hibernate is the exception — it goes through JDBC instead of CCI.

## Licensing Note

The CUBRID engine is GPL v2+, but the API drivers are separately BSD-licensed. This permits embedding the drivers in closed-source applications without GPL obligations.

## Related

- [[modules/cubrid-cci|cubrid-cci]] — the C client library that most drivers call
- [[modules/cubrid-jdbc|cubrid-jdbc]] — JDBC driver used by Hibernate
- [[modules/contrib|contrib]] — parent module page
- [[components/broker-impl|broker-impl]] — the broker/CAS tier that all drivers ultimately connect to
