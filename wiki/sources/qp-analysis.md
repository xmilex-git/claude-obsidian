---
type: source
source_type: jira-wiki-cluster
title: "QP Analysis (internal CUBRID R&D wiki)"
source_root_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-"
source_owner: "박세훈 (last edited by 채광수)"
source_last_edited: "2026-03-03"
created: 2026-05-11
updated: 2026-05-11
language: ko
domain: cubrid-query-processing
confidence: high
tags:
  - source
  - cubrid
  - query-processing
  - parser
  - optimizer
  - xasl
  - executor
status: active
related:
  - "[[Query Processing Pipeline]]"
  - "[[Data Flow]]"
  - "[[Architecture Overview]]"
  - "[[components/parser]]"
  - "[[components/semantic-check]]"
  - "[[components/optimizer]]"
  - "[[components/optimizer-rewriter]]"
  - "[[components/xasl-generation]]"
  - "[[components/query-executor]]"
sub_pages:
  - "[[qp-analysis-overview]]"
  - "[[qp-analysis-parser]]"
  - "[[qp-analysis-semantic-check]]"
  - "[[qp-analysis-rewriter]]"
  - "[[qp-analysis-optimizer]]"
  - "[[qp-analysis-xasl-generator]]"
  - "[[qp-analysis-executor]]"
  - "[[qp-analysis-tempfile]]"
---

# QP Analysis — internal CUBRID R&D wiki cluster

원본: 사내 JIRA 위키 `RND/analysis-for-query-processing-` 및 그 하위 7개 sub-page (parser, semantic check, rewriter, optimizer, xasl generator, executor, tempfile).

**작성자**: 박세훈 (개발팀). 마지막 편집: 2026-03-03 (채광수).
**언어**: 한국어 (원문 그대로 보존).
**용도**: CUBRID Query Processing 단계별 코드 분석 보고서. 신규 contributor 가이드 + 내부 코드 리딩 참조.

## 페치/보존 상태

- 본문 (HTML/text): 8개 page 본문 모두 `.raw/qp-analysis-*.md` 에 보존.
- 첨부 다이어그램: 20개 `.jpg` 모두 `_attachments/qp-analysis/` 에 보존.
- 첨부 분석 보고서 (PDF): 8개 `.pdf` (총 ~5MB) 를 `.raw/qp-pdfs/` 에 보존. 그 중 6개는 본 위키 페이지 본문에 직접 인용/요약, 2개 (Name Resolution v0.9 22p, XASL generator v1.0 23p) 는 텍스트 추출본 (`.txt`) 로 동봉.
- 첨부 슬라이드 (PPTX): Executor 0.7, Hierarchical Query, How is the query executed — 3개 `.pptx` (~2.1MB) 를 `.raw/qp-pdfs/` 에 보존, 텍스트 추출본 (`.txt`) 동봉.
- 인증: `~/.config/cubrid-skills/jira.env` + MCP playwright form-login → 같은 컨텍스트 fetch (procedure: `~/dev/cubrid/.claude/skills/cubrid-flow/procedures/jira.md`).

## CUBRID 컴포넌트 매핑

| 분석 단계 | 본 위키 page | 기존 component 페이지 | 시작 함수 |
|---|---|---|---|
| Parser | [[qp-analysis-parser]] | [[components/parser]] | `parser_main()` |
| Pre-fetch | [[qp-analysis-overview#PRE FETCH]] | (3-tier: CAS ↔ cub_server) | `pt_class_pre_fetch()` |
| Semantic check | [[qp-analysis-semantic-check]] | [[components/semantic-check]] | `pt_compile()` → `pt_check_with_info()` |
| Rewriter | [[qp-analysis-rewriter]] | [[components/optimizer-rewriter]] | `mq_translate()` |
| Optimizer | [[qp-analysis-optimizer]] | [[components/optimizer]] | `qo_optimize_query()` |
| XASL generator | [[qp-analysis-xasl-generator]] | [[components/xasl-generation]] | `pt_to_buildlist_proc()` |
| Executor | [[qp-analysis-executor]] | [[components/query-executor]] | `qexec_execute_mainblock()` |
| Tempfile (executor 부속) | [[qp-analysis-tempfile]] | — (cross-cutting) | `qmgr_create_new_temp_file()`, `file_create_temp()` |

## 키 자료구조

- `PT_NODE` — parse tree node. union `info` 가 node_type 별 다른 구조체 (`PT_SELECT_INFO`, `PT_NAME_INFO`, ...). `next`/`or_next` 로 같은 레벨 연결.
- `QO_ENV` — optimizer query graph: NODE (table), TERM (predicate), SEGMENT (column), EQCLASS (join columns), PARTITION (edge 없는 그룹).
- `QO_PLAN` — plan_type (`PLANTYPE_SCAN`/`PLANTYPE_JOIN`/`PLANTYPE_SORT`) 별 union `plan_un`.
- `QO_PLANNER` — node_info[N] (per-node best plans) + join_info[2^N] (bitmask-indexed join plans).
- `XASL_NODE` — proc_type (BUILDLIST/BUILDVALUE/SCAN/UPDATE/...). aptr/dptr/scan_ptr 3-pointer 연결.
- `REGU_VARIABLE` — XASL flexible expression container (CONSTANT/ATTR_ID/POS_VALUE/INARITH/DBVAL/...).
- `ACCESS_SPEC` + `INDEX_INFO` + `KEY_INFO` — scan spec, RANGE_TYPE (R_KEY/R_RANGE/R_KEYLIST/R_RANGE_LIST).
- `PRED_EXPR` — predicate tree (T_PRED B_OR/B_AND, T_EVAL_TERM R_EQ/R_LT/...).
- `QFILE_LIST_ID` — temp file metadata (First_vpid, Last_vpid, last_pgptr).

## 핵심 인사이트 (위키 + PDF 종합)

1. **3-tier 아키텍처와 client-side compile**: PARSER, SEMANTIC CHECK, REWRITER, OPTIMIZER, XASL generator 까지 모두 CAS (`!defined(SERVER_MODE)`) 에서 실행. cub_server 는 XASL stream 만 받는다. PRE_FETCH 가 CAS workspace 의 `Classname_cache` 에 schema 를 캐싱 (CHN: cache coherency number 로 일관성 확인). → [[Query Processing Pipeline#Side-of-line]] 와 일치.
2. **WHERE_RANGE / WHERE_KEY / WHERE_PRED 3-way 분리**: 같은 predicate 라도 인덱스 컬럼 매칭에 따라 (a) 수직 스캔 가능한 RANGE, (b) 수평 스캔 가능한 KEY, (c) 데이터 영역 평가 PRED 로 분류. `qo_analyze_term()` 에서 결정되고 XASL generator 에서 `regu_list_range/key/pred` 로 분배.
3. **CNF 변환의 인덱스 의미**: 100 항 초과시 변환 포기 → CNF 가 아닌 predicate 는 인덱스 스캔 후보가 되지 않음. DNF 로 작성된 복잡한 조건은 OPTIMIZER 의 indexable 평가 대상이 아님.
4. **first node 선별과 cost vs cardinality**: `card` 가 아닌 `cost` 비교가 우선되어, 시작 node 선정에 비효율 발생 가능 (분석서 본문에서 직접 지적). 인덱스 유무 체크가 오른쪽 분기에서만 수행됨.
5. **aptr/dptr/scan_ptr**: APTR=uncorrelated subquery (먼저 한번 실행→ temp file), DPTR=correlated subquery (매 outer row 마다 재실행), SCAN_PTR=join inner. `XASL_LINK_TO_REGU_VARIABLE` flag 가 set 된 aptr 은 일반 execute 경로에서 실행 안 되고 REGU 평가 시 한번만 실행.
6. **Tempfile 의 thread-affinity 문제 (parallel query 컨텍스트)**: `file_Tempcache.tran_files` 가 thread-local. external sort 는 preserve 임시파일 패턴으로 우회. Page FIX/UNFIX 는 동일 thread 에서. `private_alloc/private_free` 는 동일 thread 에서 free 필요 — `db_change_private_heap(thread_p, 0)` 로 일반 malloc/free 폴백.
7. **Volcano (iterator) model**: `qexec_execute_mainblock()` → pre-processing (open scan, execute APTR) → processing (`scan_next_scan()` 반복) → post-processing (group by/order by/analytic). join 으로 연결된 scan_ptr 은 mainblock 과 함께 처리, subquery (aptr/dptr) 는 독립 mainblock 으로 처리.

## 알려진 미완성 (위키 author 본인 표기)

- `qp-rewriter`: TO_DO 마커 다수 ("RewriteR 성질에 따른 구분", "작성되지 않은 주요함수들의 로직 요약").
- `qp-optimizer`: "to_do : join method, cost 산정, plan compare, plan 객체".
- `qp-resolve-names` (= semantic-check) wiki page 자체는 4개 PDF 첨부만; 실제 분석은 PDF (특히 Name Resolution v0.9 22p) 에 있음. 일부 섹션은 "TBU" 표기.

## 베이스라인 / 드리프트 메모

- CUBRID baseline: `05a7befd` (recorded in [[hot|hot cache]]).
- 현재 source HEAD: `bac5eb1615b83d1c55c1078f8d9430c5fe71275c` — baseline 의 직계 후손 아닌 divergent branch (`git merge-base --is-ancestor` 실패).
- 본 cluster ingest 는 개념-중심 (line 번호 미인용) 이라 drift 영향 최소. 함수 이름/구조체 이름은 모두 CUBRID 마스터 develop 기준이며, drift 시 함수 이름이 그대로면 호환. divergent branch 가 일반 develop 가 아닐 가능성도 있어 사용자 확인 후 baseline 갱신 미수행.

## Sub-pages

- [[qp-analysis-overview]] — 8단계 파이프라인 + Pre-fetch 3-tier 아키텍처 다이어그램
- [[qp-analysis-parser]] — PT_NODE 구조 + parser_walk_tree + Flex/Bison
- [[qp-analysis-semantic-check]] — pt_check_with_info 4단계 + name resolution + type checking + constant folding + statement-별 의미 체크
- [[qp-analysis-rewriter]] — mq_translate + qo_optimize_queries 재작성 카탈로그 (CNF, outer→inner, auto-param, ...)
- [[qp-analysis-optimizer]] — QO_ENV 빌드 → planner 객체 → node best plan → permutation
- [[qp-analysis-xasl-generator]] — PLAN → XASL_NODE 변환 + REGU_VARIABLE + ACCESS_SPEC + WHERE_KEY/RANGE/PRED
- [[qp-analysis-executor]] — qexec_execute_mainblock pre/processing/post + group-by + sub-query 실행 흐름
- [[qp-analysis-tempfile]] — temp file LIFE CYCLE + 메모리 버퍼 + 캐시 + NUMERABLE + parallel query 영향

## 원본 링크 (사내 전용, 인증 필요)

- root: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-
- parser: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-parser
- semantic check: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-resolve-names
- rewriter: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-rewriter
- optimizer: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-optimizer
- xasl generator: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-xasl-generator
- executor: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-executor
- tempfile: http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-tempfile
