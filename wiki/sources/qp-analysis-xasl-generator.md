---
type: source
source_type: jira-wiki+pdf
title: "QP Analysis — XASL Generator"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-xasl-generator"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - xasl
  - xasl-generation
status: active
related:
  - "[[qp-analysis]]"
  - "[[components/xasl-generation]]"
  - "[[components/xasl]]"
  - "[[components/xasl-cache]]"
  - "[[components/xasl-stream]]"
attachments:
  - "_attachments/qp-analysis/xasl_generator.jpg"
pdf_sources:
  - ".raw/qp-pdfs/qp_xasl_generator_v1_0.pdf (23p) — 2019-09-24 by Park Sehoon"
---

# QP Analysis — XASL Generator

> 원본 wiki 본문 없음 (PDF 첨부 1개). 본 페이지는 23-page PDF `analysis_QP_XASL_generator_1.0.pdf` (2019-09-24, 박세훈) 의 종합.

## XASL 이란?

**eXtensional Access Specification Language**. SQL 은 "어떤" 데이터를 가져올지만 명시 (순서 없음); XASL 은 PLAN 정보를 사용한 **순서가 있는 절차적 프로시져**.

### Proc Types

```
UNION_PROC       DIFFERENCE_PROC      INTERSECTION_PROC
OBJFETCH_PROC    BUILDLIST_PROC       BUILDVALUE_PROC      SCAN_PROC
MERGELIST_PROC   UPDATE_PROC          DELETE_PROC          INSERT_PROC
CONNECTBY_PROC   DO_PROC              MERGE_PROC           BUILD_SCHEMA_PROC
CTE_PROC
```

| Proc | 의미 |
|---|---|
| `BUILDLIST_PROC` | ROW 가 여러 건 될 수 있는 SQL (대부분의 SELECT). temp list 사용. |
| `BUILDVALUE_PROC` | ROW 가 한 건만 (예: `select sum(col1) from tab`). |
| `SCAN_PROC` | TEMP 영역 사용 X. join 의 inner scan 등에 사용. |

## PLAN → XASL 변환 다이어그램

![[_attachments/qp-analysis/xasl_generator.jpg]]

**해석** — `select 1 from A,B,C ... join order: A → B → C`:

좌측 PLAN tree:
```
PLAN_JOIN (plan_un.join.outer=… inner=PLAN_SCAN TAB C)
└── PLAN_JOIN (outer=PLAN_SCAN TAB A, inner=PLAN_SCAN TAB B)
```

우측 XASL chain:
```
XASL_NODE BUILDLIST tab a  (aptr/dptr/scan_ptr)
   scan_ptr → XASL_NODE SCAN_PROC tab b
                 scan_ptr → XASL_NODE SCAN_PROC tab c
```

알고리즘 (PDF page 22-23):

```c
gen_outer(env, plan):
  case QO_PLANTYPE_SCAN:
    scan_ptr = inner_scan
    add_access_spec(env, plan)         // spec list 생성
    add_scan_proc(inner_scans)         // scan_ptr 생성
    add_subqueries                     // aptr / dptr 생성
  case QO_PLANTYPE_JOIN:
    outer_plan = plan->plan_un.join.outer
    inner_plan = plan->plan_un.join.inner
    inner_scan = gen_inner(inner_plan)
    gen_outer(inner_scan)              // 재귀
```

전체 진입:

```c
pt_to_buildlist_proc:
  pt_to_outlist(select.list)           // outlist 생성
  pt_set_aptr(select)                  // aptr (uncorrelated subquery) 생성
  pt_gen_optimized_plan(select, plan)  // 나머지
    qo_to_xasl(plan) → gen_outer(env, plan)
  pt_set_dptr(select.list)             // dptr (correlated subquery) 생성
```

## XASL_NODE 의 3가지 포인터

```
SELECT (select col1 from tab c where col1 = b.col1)        ── DPTR
FROM tab a, tab b
WHERE a.col2 = b.col2
  AND a.col1 = (select col1 from tab d)                    ── APTR
-- join 순서: a → b                                          ── SCAN_PTR
```

| 포인터 | 의미 |
|---|---|
| `aptr` | uncorrelated subquery. 한 번 실행되어 결과를 temp 에 저장 후 재사용 |
| `dptr` | correlated subquery. 매 outer row 마다 재실행 (반드시 SCAN 단계에 들어감) |
| `scan_ptr` | join 의 inner. `outer.scan_ptr → inner` chain. mainblock 과 함께 실행 |

EXECUTE 순서:
```
SCAN aptr            (한번)
SCAN node            (outer)
for each row of node:
   SCAN dptr         (correlated; row 마다)
   SCAN scan_ptr     (join inner)
```

## XASL_NODE 의 필드 (BUILDLIST 기준)

```
XASL_NODE TYPE = BUILDLIST
├── outptr_list    ← select list 의 컬럼/상수 → REGU_VAR_LIST
├── var_list       ← 테이블의 컬럼 list → DB_VALUE_LIST (outptr 과 동일 컬럼이면 DB_VALUE 공유)
├── spec_list      ← access spec (class oid, index info, predicate)
├── aptr / dptr / scan_ptr
```

## REGU_VARIABLE — 만능 expression 컨테이너

| TYPE | 의미 |
|---|---|
| `CONSTANT` | DB_VALUE 직접 보유 (`dbval`) |
| `ATTR_ID` | column id + cache_dbval (테이블 컬럼 접근) |
| `POS_VALUE` | val_pos (positional reference; KEY_RANGE 의 key1/key2 에서 자주 등장) |
| `INARITH` | `arithptr` 가 `ARITH_NODE` 가리킴 (op + leftptr/rightptr/thirdptr, 예: `nvl(col1,0)`) |
| `DBVAL` | dbval 포인터 (PT_VALUE 출신) |
| 그 외 | funcp, dbvalptr… |

`fetch_peek_dbval()` 가 REGU → DB_VALUE 추출. EXECUTOR 단계의 모든 데이터 읽기/쓰기 단위.

## ACCESS_SPEC 구조

```
ACCESS_SPEC type=CLASS, access=INDEX
├── indexptr  → INDEX_INFO (btree id, RANGE_TYPE, KEY_INFO)
├── where_key
├── where_pred
├── where_range
└── s(scan node) → HYBRID_NODE (CLASS / SET / JSON / LIST) → CLS_SPEC_NODE
                    ├── regu_list_key
                    ├── regu_list_pred
                    ├── regu_list_rest
                    ├── regu_list_range
                    ├── output_val_list
                    └── val_list
```

### RANGE_TYPE

| 값 | 의미 |
|---|---|
| `R_KEY` | `=` |
| `R_RANGE` | `>` / `<` |
| `R_KEYLIST` | `IN (...)` |
| `R_RANGE_LIST` | `> OR <` 합집합 |

`KEY_INFO` 가 `key_cnt`, `is_constant`, `key_range[]`. 각 `KEY_RANGE` 에 `range = EQ_NA / LT_NA / ...` 와 `key1` (lower limit), `key2` (upper limit). `BETWEEN 1 AND 2` 면 key1=1, key2=2.

### WHERE_RANGE / WHERE_KEY / WHERE_PRED — 3-way 분리

```
INDEX(col1, col2, col3)
SELECT * FROM tbl WHERE col1=1 AND col3=2 AND col4=2

→ col1=1   → WHERE_RANGE  (인덱스 수직 스캔 가능)
→ col3=2   → WHERE_KEY    (인덱스 수평 스캔 가능)
→ col4=2   → WHERE_PRED   (데이터 영역에서 평가)
```

PT_NODE 의 predicate 가 가벼운 **`PRED_EXPR`** 구조로 변환됨 (TYPE: `T_PRED` with op `B_OR`/`B_AND`, `T_EVAL_TERM` with op `R_EQ`/`R_LT`/…).

### CLS_SPEC_NODE — regu_list 4종

`range/key/pred/rest` 로 컬럼 접근 동선 분리:
- `regu_list_range` — RANGE 스캔에서 사용할 컬럼들의 REGU_VAR
- `regu_list_key` — KEY 스캔에서 사용
- `regu_list_pred` — PRED 평가에서 사용
- `regu_list_rest` — range + key (인덱스로 이미 알아낸 컬럼들; data 영역 재읽기 회피)
- `output_val_list` — 해당 spec 이 **writer** 로서 채울 컬럼 list (`OUTPTR_LIST`)
- `val_list` — `rest` 에 대응하는 **reader** list

> 동일 컬럼의 REGU_VAR 은 같은 DB_VALUE 를 공유 → 한번의 fetch 로 여러 평가 단계에서 재사용.

## SCAN 실행 의사코드 (single table)

```c
for index scan using KEY_INFO(1, 2):
  fetch row from OID
  for each row:
    evaluate PRED_EXPR(col3 = 3)
    if qualified: end_one_iteration
```

## Cross-references

- [[components/xasl-generation]] — plan-to-xasl 변환 함수들
- [[components/xasl]] — XASL_NODE 핵심 자료구조
- [[components/xasl-cache]] — XASL plan cache (auto-param 과 연계)
- [[components/xasl-stream]] — XASL 직렬화 (client → server 전송)
- [[qp-analysis-optimizer]] — PLAN 생성 단계
- [[qp-analysis-executor]] — XASL 실행 단계
- 원문 PDF 보존: `.raw/qp-pdfs/qp_xasl_generator_v1_0.pdf` + 추출본 `.txt`
