---
type: concept
title: "XASL_NODE"
status: stub
domain: cubrid-xasl
aliases: ["XASL plan node", "xasl_node"]
created: 2026-05-11
updated: 2026-05-11
tags:
  - concept
  - cubrid
  - xasl
related:
  - "[[components/xasl]]"
  - "[[components/xasl-generation]]"
  - "[[components/query-executor]]"
  - "[[sources/qp-analysis-xasl-generator]]"
  - "[[sources/qp-analysis-executor]]"
---

# XASL_NODE

**eXtensional Access Specification Language** 의 plan node. CUBRID 의 직렬화-가능 실행 계획 IR — client (CAS) 에서 컴파일 후 server (cub_server) 로 stream 전송됨. 자세한 자료구조는 [[components/xasl]], 생성 알고리즘은 [[components/xasl-generation]], 실행은 [[components/query-executor]].

## Proc Type (16종)

`UNION_PROC / DIFFERENCE_PROC / INTERSECTION_PROC / OBJFETCH_PROC / BUILDLIST_PROC / BUILDVALUE_PROC / SCAN_PROC / MERGELIST_PROC / HASHJOIN_PROC / UPDATE_PROC / DELETE_PROC / INSERT_PROC / CONNECTBY_PROC / DO_PROC / MERGE_PROC / BUILD_SCHEMA_PROC / CTE_PROC`

대표:
- `BUILDLIST_PROC` — ROW 여러 건 (대부분의 SELECT). temp list 사용.
- `BUILDVALUE_PROC` — ROW 한 건 (예: aggregate-only).
- `SCAN_PROC` — TEMP 영역 X. join inner.

## 3가지 연결 포인터 (실행 시간 의미)

| 포인터 | sub-query 종류 | EXECUTE 빈도 | 결과 처리 |
|---|---|---|---|
| `aptr_list` | uncorrelated subquery / CTE | 1회 (pre-processing) | temp list 캐싱 |
| `dptr_list` | correlated subquery | outer row 마다 | 매 평가 즉시 소비 |
| `scan_ptr` (proc 내부) | join inner | mainblock 동행 | 1-row 단위 |

`XASL_LINK_TO_REGU_VARIABLE` flag set 시 aptr 의 일반 execute 경로 우회 → REGU 평가 시 1회. 자세히 [[sources/qp-analysis-executor]].

## 주요 필드 요약

```c
struct xasl_node {
  XASL_NODE_HEADER  header;
  XASL_NODE        *next;
  PROC_TYPE         type;        // 16 종
  QFILE_LIST_ID    *list_id;
  OUTPTR_LIST      *outptr_list;
  ACCESS_SPEC_TYPE *spec_list;   // table/index/list/json/dblink scan
  PRED_EXPR        *during_join_pred / after_join_pred / instnum_pred;
  XASL_NODE        *aptr_list / dptr_list;
  union {...} proc;              // PROC_TYPE 별 활성
};
```

## See also

- [[concepts/PT_NODE]] — 입력 (xasl_generation 이 PT_NODE → XASL_NODE)
- [[concepts/QO_ENV]] — PLAN 의 source (QO_PLAN → XASL_NODE via gen_outer/gen_inner)
