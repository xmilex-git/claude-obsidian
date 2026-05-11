---
type: source
source_type: jira-wiki+pptx
title: "QP Analysis — Executor"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-executor"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - executor
  - volcano-model
  - iterator
status: active
related:
  - "[[qp-analysis]]"
  - "[[components/query-executor]]"
  - "[[components/query-evaluator]]"
  - "[[components/query-fetch]]"
  - "[[components/query-manager]]"
attachments:
  - "_attachments/qp-analysis/executor.jpg"
pptx_sources:
  - ".raw/qp-pdfs/qp_executor_v0_7.pptx (19 slides) — 2020-03-16"
  - ".raw/qp-pdfs/qp_hierarchical_query.pptx (12 slides)"
  - ".raw/qp-pdfs/how_is_the_query_executed.pptx (13 slides)"
---

# QP Analysis — Executor

> 원본 wiki 본문 없음 (PowerPoint 3개 첨부). 본 페이지는 `qp_executor_v0_7.pptx` (19 slides, 2020-03-16) 의 종합 + executor 이미지 해석.

## Volcano (Iterator) Model

`qexec_execute_mainblock()` 가 진입점. 3-phase:

```
Pre-processing
  qexec_open_scan()            (main XASL + scan_ptr)
  qexec_execute_mainblock(APTR)
  allocate_agg_hash_context()

Processing                       ← 행 한 줄씩 산출
  qexec_intprt_fnc()
  qexec_next_scan_block_iterations()
    while scan_next_scan():
      qexec_execute_mainblock(DPTR)
      qexec_execute_scan(SCAN_PTR)
      qexec_end_one_iteration()
  qexec_end_mainblock_iterations()

Post-processing
  qexec_groupby()
  qexec_execute_analytics()
  qexec_orderby_distinct()
```

## XASL 실행 다이어그램

![[_attachments/qp-analysis/executor.jpg]]

**해석** — 쿼리:

```sql
SELECT (select col1 from tab c where col1 = b.col1)   -- DPTR (correlated)
FROM tab a, tab b                                       -- main + scan_ptr (join)
WHERE a.col2 = b.col2
  AND a.col1 = (select col1 from tab d)                -- APTR (uncorrelated)
```

XASL tree:
```
XASL_NODE BUILDLIST tab a   (보라색)
  aptr → XASL_NODE BUILDLIST tab d   (빨강; SELECT clause 의 select col1 from tab d 가 식별 안 됨에 주의 — 실제로는 WHERE 의 uncorrelated 가 aptr)
  scan_ptr → XASL_NODE SCAN_PROC tab b   (초록; join inner)
    dptr → XASL_NODE BUILDLIST tab c    (보라; correlated)
```

EXECUTE 흐름:
```
SCAN aptr            // tab d 결과를 temp 에 한번
SCAN node (tab a)
for each row of tab a:
  SCAN dptr          // tab c with col1 = b.col1, row 마다
  SCAN scan_ptr      // tab b (join)
```

## Processing 의 분기 (slide 8-9)

`scan_ptr` 는 mainblock 과 함께 처리됨 — `execute_mainblock` 안에서 `SCAN node → for each row → SCAN scan_ptr`. 반면 `aptr/dptr` 의 subquery 는 **다른 main block 으로 독립 실행** — `execute_mainblock(aptr)` / `execute_mainblock(dptr)` 가 별도 호출됨.

### `scan_next_scan()` 의 상태머신

```
qexec_intprt_fnc()
  FOR scan_next_scan():
    Get 1 row?
    SUCCESS → FOR Scan(scan_ptr)
                Next_scan_on?
                  NO  → END
                  YES → Qualified? → return SUCCESS / END
    END → post-processing
  YES → Write 1row → end_one_iteration() → next iteration
```

반환값:
- `SUCCESS` — predicate 까지 적용된 1 row 가 만들어진 상태
- `END` — 더 이상 조회될 데이터 없음

### Block iteration (slide 10)

ALL super_class 조회 시 (상속/inheritance):

```sql
create table super_class (col1 int);
create table sub_class under super_class;
select * from ALL super_class a, ALL super_class b where a.col1 = b.col1;
```

`(a∪b) ⋈ (c∪d) = (a⋈c) ∪ (a⋈d) ∪ (b⋈c) ∪ (b⋈d)` — block iteration 이 next access spec 을 처리하기 위해 각 XASL block 을 초기화함 → 4가지 조합을 차례로.

## Sub-query 실행 메커니즘 (slide 11-12)

### Select / WHERE 의 subquery → REGU_VARIABLE 평가 시 실행

```sql
SELECT (select col1 from tab a where col1 = b.col1) FROM tab b
```

select 또는 where 의 subquery 는 REGU_VAR 에 담기고 (`fetch_peek_dbval` 흐름) 그 변수를 가져오는 과정에서 실행된다.

- **uncorrelated**: `XASL_LINK_TO_REGU_VARIABLE` flag 가 set 되면 기존 executor 의 일반 수행 경로에서는 실행되지 않고 REGU 평가 시 한 번만 수행 → REGU_VAR 가 초기화되지 않음으로 캐싱 효과.
- **correlated**: 매 outer row 마다 평가.

```
SCAN tab b
  scan_next_scan
  end_one_iteration
  fetch_peek_dbval  ── execute(tab a)
```

### FROM 의 subquery → APTR + temp list scan (slide 12)

```sql
SELECT a.col1
FROM (select col1 from tab a where col2=3) a, tab b
WHERE a.col1 = b.col1
```

FROM 절의 derived table 은 APTR 으로 수행되어 결과가 temp file 에 담기면 그것을 읽어서 처리:

```
Pre-processing
  Execute aptr (tab a)
Processing
  SCAN temp list scan
  scan_next_scan
    scan scan_ptr (tab b)
  end_one_iteration
```

## 1-row Scan 내부 (slide 13)

```
scan_next_scan()
  scan_next_heap_scan() (heap scan, index scan, list scan 등)
    heap_next()                    // Read data from heap
    eval_data_filter()             // predicate 필터
    heap_attrinfo_read_dbvalues()  // Read attribute from record
    fetch_val_list()               // Fetch regu_var from attr_info
    eval_fnc()                     // Evaluate expressions
    heap_attrinfo_read_dbvalues()
    fetch_val_list()
end_one_iteration()
  qexec_generate_tuple_descriptor()
  → write 1 row to temp file
```

> Predicate 사용 컬럼 vs 나머지를 구분하여 데이터를 regular variable 에 넣는다 (= predicate 평가에 필요한 최소 컬럼만 먼저 fetch → 조건 fail 시 나머지 fetch 회피).

## Post-processing (slide 14-17)

### Group by (`qexec_groupby()`)

- Aggregate function (`SUM/AVG/COUNT/...`) 은 한 그룹당 한 결과.
- 두 구현:
  - **Sort group by**: scan → result list → sort → 정렬 list 를 순회하며 grouping → evaluation
  - **Hash group by**: scan 중 in-memory hash ACC 에 누적 → memory overflow 시 partial list 로 spill → 모든 row 처리 후 partial list 들 sort + merge

Sort group by 함수:
- `qexec_gby_get_next()`, `qexec_gby_put_next()`
- Evaluation ACC: `qdata_evaluate_aggregate_list()`
- Replace Hash ACC: `qdata_load_agg_hvalue_in_agg_list()`

Hash group by 함수:
- `qexec_hash_gby_get_next()`, `qexec_hash_gby_put_next()`
- `qexec_hash_gby_agg_tuple()` (`end_one_iteration` 후)
- `qdata_aggregate_accumulator_to_accumulator()` (ACC-ACC 합치기)
- `qdata_save_agg_htable_to_list()` (overflow spill)

ACCUMULATOR: 집계함수 1개당 1개 생성. HASH ACUUMULATOR: 집계함수 갯수만큼 배열.

### Order by / Distinct (`qexec_orderby_distinct()`)

`sort_listfile()` 에 `SORT_DUP` 옵션으로 처리:
- `SORT_ELIM_DUP` — duplicate 제거 (DISTINCT)
- `SORT_DUP` — duplicate 유지

### Analytic functions (`qexec_execute_analytics()`)

`LAG/RANK/...`. PDF 본문은 "분석필요" 로 표기 (미완성).

## Cross-references

- [[components/query-executor]] — 기존 component 페이지
- [[components/query-evaluator]] — `eval_fnc()` 등 predicate evaluation
- [[components/query-fetch]] — `fetch_*` API
- [[components/query-manager]] — `qmgr_*` (temp file 관련)
- [[qp-analysis-xasl-generator]] — XASL 생성 단계
- [[qp-analysis-tempfile]] — group by/sort/derived 가 모두 temp file 사용
- 원본 PPTX 보존: `.raw/qp-pdfs/qp_executor_v0_7.pptx` + 추출본 `.txt`
- 보조 PPTX: `qp_hierarchical_query.pptx`, `how_is_the_query_executed.pptx`
