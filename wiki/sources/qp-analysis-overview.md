---
type: source
source_type: jira-wiki
title: "QP Analysis — Overview (pipeline + pre-fetch)"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - query-processing
status: active
related:
  - "[[qp-analysis]]"
  - "[[Query Processing Pipeline]]"
  - "[[Data Flow]]"
  - "[[components/parser]]"
  - "[[components/semantic-check]]"
  - "[[components/optimizer]]"
  - "[[components/optimizer-rewriter]]"
  - "[[components/xasl-generation]]"
  - "[[components/query-executor]]"
attachments:
  - "_attachments/qp-analysis/query_process.jpg"
  - "_attachments/qp-analysis/pre_fetch.jpg"
---

# QP Analysis — Overview (8단계 파이프라인)

> 원본: [analysis-for-query-processing-](http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-)
> 본 페이지는 hub 페이지 [[qp-analysis]] 의 첫번째 sub-page. 8단계 QP 파이프라인 전체 그림을 한 페이지에 요약한다.

## 파이프라인 마스터 다이어그램

![[_attachments/qp-analysis/query_process.jpg]]

**해석** (직접 다이어그램에서 추출):

```
QUERY
  → PARSER          parser_main()
    → PT_NODE
  → PRE_FETCH       pt_class_pre_fetch()
  → SEMANTIC_CHECK  pt_compile()
    → PT_NODE with info
  → REWRITER        mq_translate()
    → Optimized PT_NODE
  → OPTIMIZER       qo_optimize_query()
    → plan
  → XASL generator  to_buildlist_proc()
    → XASL
  → EXECUTOR        qexec_execute_mainblock()
    → RESULT
```

각 화살표는 (1) 자료구조 변환 시점이자 (2) 책임 모듈 경계이다.

## 단계별 1줄 요약 (분석서 머리말)

1. **PARSER** — SQL 텍스트를 BISON 으로 파싱 → `PT_NODE` 트리 생성. 동시에 syntax check + 일부 사소한 rewrite.
2. **PRE_FETCH** — 3-tier 아키텍처 특유의 단계. CAS 가 cub_server 로부터 테이블 schema/meta 를 가져와 workspace 에 caching. CHN (cache coherency number) 으로 객체 최신성 확인.
3. **SEMANTIC_CHECK** — (a) name resolution: `PT_NAME` 들을 `PT_SPEC` 과 binding, (b) type checking: 인수 타입 호환 + 필요 시 CAST 삽입, (c) constant folding: 상수식 미리 계산.
4. **REWRITER** — `mq_translate()` 진입. 성능 향상 + optimizer 가 다룰 수 있는 형태로 parse tree 재작성. CNF 변환, view→derived table, predicate push down, auto-parameterize 등.
5. **OPTIMIZER** — `qo_optimize_query()`. `QO_ENV` 빌드 후 `QO_PLANNER` 로 join 순서/method/index 조합 비교. cost-based.
6. **XASL generator** — PLAN + parse tree 종합 → `XASL_NODE` 트리 생성. aptr/dptr/scan_ptr 3가지 포인터로 연결.
7. **EXECUTOR** — Volcano model. `qexec_execute_mainblock()` 으로 XASL 실행 → result.
8. **(부속)** Tempfile — SELECT 결과/GROUP BY-ORDER BY/subquery 결과/index build 결과 저장. 메모리 버퍼 우선 → 임시파일 fallback.

## Pre-fetch — 3-tier 캐싱 아키텍처

![[_attachments/qp-analysis/pre_fetch.jpg]]

**해석**: CUBRID 의 3-tier 구조 (Cub_server ↔ CAS ↔ Client) 에서 schema 정보 동기화. 좌측 cub_server 가 object locator / Statistics / Transaction manager 를 보유. 가운데 CAS 가 SEMANTIC CHECK (`pre_fetch`, `resolve_name`) 와 OPTIMIZE (`get_statistics`) 단계에서 cub_server 와 통신. CAS 의 **workspace** 가 캐시:

- **Classname_cache**: 클래스명 → Class object 의 hash table. `Sm_class` (type, owner, collation, ..., statistics) 가 저장됨.
- **Ws_commit_mops**: 트랜잭션이 참조한 object 들의 linked list of `MOP` (memory object pointer). commit 시 entry 제거, rollback 시 관련 object 를 decache.

원본 분석서의 지적:

> rollback 시 cache 된 object 를 삭제하는 것은 DDL 을 통해 CAS 의 저장된 object 만 변경이 되어 있을 수 있기 때문이다. … 하지만 DDL 이 없는 경우에도 CAS 의 object 를 decache 하는 점은 비효율적인 로직이라 생각된다.

이는 향후 client-side 부하 분산 최적화의 후보 지점.

## CUBRID 3-tier 의 client-side compile

> [!key-insight] PARSER ~ XASL_generator 는 모두 CAS 에서 실행
> XASL stream 만 cub_server 로 전송된다 (`#if !defined(SERVER_MODE)` 가드). 즉 XASL 의 존재 이유는 **직렬화 가능한 IR** 이라는 점에 있다. CAS 가 무거운 컴파일을 부담하고 server 는 실행에만 집중. → [[Query Processing Pipeline#Side-of-line]] 와 동일.

## Cross-references

- [[Query Processing Pipeline]] — 기존 concept 페이지 (요약 + side-of-line)
- [[Data Flow]] — 큐브리드 전체 데이터 흐름
- 단계별 상세: [[qp-analysis-parser]] [[qp-analysis-semantic-check]] [[qp-analysis-rewriter]] [[qp-analysis-optimizer]] [[qp-analysis-xasl-generator]] [[qp-analysis-executor]] [[qp-analysis-tempfile]]
- 원문 한국어 본문은 `.raw/qp-analysis-overview.md` 에 그대로 보존.
