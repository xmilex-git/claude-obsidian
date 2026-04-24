---
type: module
path: "pl_engine/"
status: active
language: "Java"
build: "Gradle"
purpose: "Java PL (Procedural Language) server for stored procedures"
last_updated: 2026-04-23
depends_on: []
used_by:
  - "[[components/sp|src/sp (JNI bridge)]]"
tags:
  - module
  - cubrid
  - java
  - stored-procedure
related:
  - "[[CUBRID]]"
  - "[[components/sp|src/sp]]"
  - "[[Architecture Overview]]"
created: 2026-04-23
updated: 2026-04-23
---

# `pl_engine/` — Java PL Engine

Java-based stored procedure server. Built with **Gradle**. Bridged to the C++ engine via JNI through [[components/sp|src/sp/]].

## Why a separate Java process

Stored procedures in CUBRID are written in Java. The PL engine hosts the JVM and executes those procedures. The C++ engine invokes them through the SP JNI bridge.

## Build

- Gradle wrapper expected (`gradlew`)
- Output: PL server executable / JAR consumed by the engine
- Coordinates with the main CMake build via `build.sh`

## Where SP feature work spans

Per [[cubrid-AGENTS|AGENTS.md]] "Where to look" table:

> Add stored procedure feature → `src/sp/` + `pl_engine/`

So a single SP feature touches **both** the JNI bridge in C++ and the Java executor here.

## See also

- `pl_engine/AGENTS.md` exists with deeper detail (separate ingest)
- [[components/sp]] (JNI bridge)
- [[components/method]] (method invocation from queries — adjacent concept)
