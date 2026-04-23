---
type: index
title: "CUBRID Components"
updated: 2026-04-23
tags:
  - index
  - cubrid
  - component
status: active
---

# Components

Reusable / functional components inside CUBRID modules. One page per subsystem or meaningful abstraction (e.g. query optimizer, lock manager, page buffer, log recovery, HA replication).

Navigation: [[index]] | [[modules/_index|Modules]] | [[Architecture Overview]]

---

## Query Execution Layer (`src/query/`)

- [[components/query|query]] — hub page: full XASL execution layer overview
- [[components/query-executor|query-executor]] — `qexec_execute_mainblock`, XASL dispatch, hash GROUP BY
- [[components/scan-manager|scan-manager]] — unified scan abstraction (15 scan types)
- [[components/cursor|cursor]] — client-side result cursor over QFILE_LIST_ID
- [[components/partition-pruning|partition-pruning]] — runtime partition elimination and DML routing
- [[components/dblink|dblink]] — remote CUBRID query (CCI) + 2-phase commit
- [[components/list-file|list-file]] — temp result spool, sort, set ops, result cache
- [[components/aggregate-analytic|aggregate-analytic]] — GROUP BY / window functions
- [[components/filter-pred-cache|filter-pred-cache]] — compiled filter predicate cache (filtered indexes)
- [[components/memoize|memoize]] — subquery result memoization

## Parallel Query (`src/query/parallel/`)

- [[components/parallel-query|parallel-query]] — parallel execution subsystem hub
- [[components/parallel-worker-manager|parallel-worker-manager]] — pool lifecycle and reservation
- [[components/parallel-task-queue|parallel-task-queue]] — MPMC queue and callable_task
- [[components/parallel-hash-join|parallel-hash-join]] — hash join parallelism
- [[components/parallel-heap-scan|parallel-heap-scan]] — heap scan parallelism
- [[components/parallel-query-execute|parallel-query-execute]] — subquery parallelism
- [[components/parallel-sort|parallel-sort]] — external sort parallelism

## Parser (`src/parser/`)

- [[components/parser|parser]] — SQL → parse tree (PT_NODE) → XASL generation
