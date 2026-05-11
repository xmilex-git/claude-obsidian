---
type: source
source_type: jira-wiki
title: "QP Analysis — Optimizer"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-optimizer"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - optimizer
  - plan
  - join-order
status: active
related:
  - "[[qp-analysis]]"
  - "[[components/optimizer]]"
  - "[[components/xasl-generation]]"
attachments:
  - "_attachments/qp-analysis/optimizer1.jpg"
  - "_attachments/qp-analysis/qo_env.jpg"
  - "_attachments/qp-analysis/optimizer_pre_info.jpg"
  - "_attachments/qp-analysis/optimizer.jpg"
  - "_attachments/qp-analysis/optimizer_planner.jpg"
  - "_attachments/qp-analysis/optimizer_planner2.jpg"
  - "_attachments/qp-analysis/optimizer_planner3.jpg"
  - "_attachments/qp-analysis/optimizer_planner4.jpg"
  - "_attachments/qp-analysis/optimizer_planner5.jpg"
  - "_attachments/qp-analysis/optimizer_planner6.jpg"
---

# QP Analysis — Optimizer

> 원본: [analysis-for-qp-optimizer](http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-optimizer)
> OPTIMIZER 는 **어떻게 (HOW) DATA 를 검색할지** 결정. SQL 은 **어떤 (WHAT)** 만 명세하므로 cost-based 로 join 순서 / join method / index 조합을 탐색.

## 2-단계 구조

![[_attachments/qp-analysis/optimizer1.jpg]]

**해석**: optimizer 박스 하나가 사실은 두 단계.

1. **사전 정보 생성** — `optimize_helper()` 가 `QO_ENV` 구조체 빌드
2. **Optimization 수행** — `qo_planner_search()` 가 `qo_planner` 객체로 plan 비교 → best plan

진입 콜체인:

```
parser_generate_xasl
  parser_walk_tree (post-order, parser_generate_xasl_post)
    parser_generate_xasl_proc
      pt_plan_query
        qo_optimize_query        ← optimization
        pt_to_buildlist_proc     ← xasl generation (다음 단계)
```

post-order 순회이므로 sub-query 의 PLAN/XASL 이 먼저 생성된 뒤 상위에 연결됨.

## 주요 용어 (CUBRID 고유 어휘)

| 용어 | 일반 RDBMS 대응 |
|---|---|
| **SEGMENT** | column |
| **final segment** | select list 의 컬럼 (bit set 으로 표현) |
| **NODE** | table (PT_SPEC) |
| **TERM** | predicate (조회조건 + 조인조건) |
| **edge** | join term (term 중 join 에 해당) |
| **PARTITION** | edge 없는 독립 node 그룹 (cartesian product 로 따로 합쳐짐) |
| **EQCLASS** | join term 들의 segment 모음 (sort-merge join 의 sort column 결정) |

## `QO_ENV` 구조 — parse tree 와의 매핑

![[_attachments/qp-analysis/qo_env.jpg]]

**해석**: 단순 SELECT `... where col1=1 and col2=1` 에서:

- SEGMENT: `col1`, `col2`. final segment bit set: `0011` (둘 다 select list 사용)
- TERM: `col1=1`, `col2=1` (둘 다 single-spec → QO_TC_SARG)
- NODE: `tbl a` (단일)
- Partition: none
- Eqclass: none
- Sub query: none

> 디버깅 팁: `set optimization level 513;` 으로 plan 출력 상단에 QO_ENV 텍스트 dump 가 함께 나옴.

## TERM CLASS bit encoding

```
TERM CLASS              VALUE  PATH  EDGE  FAKE  NUM
QO_TC_PATH              0x30    1     1     0    000   ORDBMS PATH join
QO_TC_JOIN              0x11    0     1     0    001   일반 join predicate
QO_TC_DEP_LINK          0x1c    0     1     1    100   correlated subquery linkage
QO_TC_DEP_JOIN          0x1d    0     1     1    101
QO_TC_DUMMY_JOIN        0x1f    0     1     1    111   sql-표준 outer join 보조
QO_TC_SARG              0x02    0     0     0    010   일반 조회조건
QO_TC_OTHER             0x03    0     0     0    011
QO_TC_DURING_JOIN       0x04    0     0     0    100   outer join on 절
QO_TC_AFTER_JOIN        0x05    0     0     0    101   outer join where 절
QO_TC_TOTALLY_AFTER_JOIN 0x06   0     0     0    110
```

비트 자릿수: **PATH** (1=indirect relation, ORDBMS 컬렉션) / **EDGE** (1=join 관련) / **FAKE** (1=가상 term, 실제 predicate 없음).

`qo_analyze_term()` 에서 모든 term 의 class 평가 + selectivity + indexable 여부 (op/LHS/RHS type 만으로 1차 판정; 실제 인덱스 매칭은 후속) 저장.

### Outer join 관련 TERM CLASS

```sql
from tbl1 a left outer join tbl2 b on  a.col1 = b.col1 and b.col2 = 2  -- QO_TC_DURING_JOIN
where nvl(b.col3,1) = 4                                                 -- QO_TC_AFTER_JOIN
```

- `on` 절의 sarg 류 predicate = `QO_TC_DURING_JOIN` (join 시 평가, 조건 false 여도 outer row 반환)
- `where` 절의 sarg 류 predicate = `QO_TC_AFTER_JOIN` (join 결과에 대해 평가)

null 비허용 predicate 라면 rewriter 의 `qo_rewrite_outerjoin()` 단계에서 이미 inner 로 변환됨 → 여기까지 오면 nullable.

## 사전 정보 생성 — `optimize_helper()` 흐름

![[_attachments/qp-analysis/optimizer_pre_info.jpg]]

**해석** (위→아래 진행):

1. `build_query_graph()` — node + segment 추가 (`qo_add_node`, statistics 의 NCARD/TCARD 저장; derived table 은 `xasl->cardinality`)
2. **dep_set / dep_term 추가** — correlated subquery / set-type spec / json table → `QO_TC_DEP_LINK` FAKE TERM 생성 (`qo_add_dep_term`)
3. **term, dummy term 추가** — ON_COND term 추가 + WHERE predicate term 추가 (`qo_add_term`/`qo_analyze_term`). sql-표준 outer join 의 연속 node 사이에 join term 없으면 dummy term 추가 (`qo_add_dummy_join_term`)
4. **outer join term class 설정** — `qo_classify_outerjoin_terms()` 가 DURING/AFTER 구분 + `node.outer_dep_set` (permutation 제약) 설정
5. **edge, sargs term 정렬** — `qo_discover_edges()` 가 edge term 을 sarg 앞에 두고 selectivity 내림차순. NODE.sargs 의 bitset 채움. selectivity = node card × sarg selectivity 곱
6. **index entry 정보 생성** — `qo_discover_indexes()` 가 SARG class term 들과 index 컬럼 매칭. `index_entry->seg_equal_terms[idx]` (= 또는 RANGE LIST 한번) vs `seg_other_terms[idx]` (그 외)
7. **partition** — `qo_discover_partitions()` 가 edge 없는 node 그룹 분리

### ENV 초기화 핵심

`qo_env_init()` 에서:
- node 갯수 **64** 초과시 에러 (cubrid 의 절대 한계)
- `qo_validate()` 가 parse tree 로부터 node/term/segment 갯수 예측
- `Nnodes/Nsegs/Nterms/Neqclasses` 는 최대 할당 자리수, `nnodes/nsegs/nterms/neqclasses` 는 실제 할당 갯수

### DEPENDENT TABLE 케이스

- `PT_IS_SET_EXPR` (예: `from tbl a, TABLE(a.col1) b(x)`)
- `PT_IS_CSELECT` (제거된 MERGE/CSELECT 잔재; 미사용)
- `PT_DERIVED_JSON_TABLE`
- `PT_IS_SUBQUERY && correlation_level == 1` (correlated subquery)

이 경우 scan 순서가 강제됨 (외부 참조 node 가 먼저 와야 함). `subquery.dep_set` 에 연관 NODE 의 bitset 저장 + `QO_TC_DEP_LINK` FAKE TERM 생성. permutation 에서 강한 제약으로 작용.

### Final segment / EQ_CLASS

- `qo_add_final_segment()` — select_list, group by, having, connect by 의 segment 들. group by 등은 executor 의 후처리이므로 final 에 포함.
- `qo_assign_eq_classes()` — join term 의 양변 segment 들을 같은 eq class 로. sort merge join 의 sort key 후보 식별에 사용.

## Plan 객체 — `QO_PLAN`

![[_attachments/qp-analysis/optimizer.jpg]]

**해석**: `select 1 from a,b where a.col1=b.col1 and a.col2=1 and b.col2=1 order by a.col1`, `index(b.col1)`.

```
PLANTYPE_SORT   (sort_type = SORT_ORDERBY, subplan ↓)
  PLANTYPE_JOIN (join_method = idx_join, outer/inner ↓)
    PLANTYPE_SCAN (a)  scan_method = seq_scan, term=0, kf_term=0, qo_node*
    PLANTYPE_SCAN (b)  scan_method = idx_scan, term=1, kf_term=0
```

| `plan_type` | `plan_un` 내용 |
|---|---|
| `PLANTYPE_SCAN` | scan_method, term (key range), kf_term (key filter), sarg term (data filter) 셋으로 분리된 인덱스 term들, qo_node |
| `PLANTYPE_JOIN` | outer / inner / join_method (nl-join / idx_join / sort-merge) |
| `PLANTYPE_SORT` | sort_type (ORDERBY / …) + subplan |

PT_NODE 의 `info` union 패턴과 동일 — plan_type 이 plan_un 의 활성 멤버를 결정.

## `QO_PLANNER` 객체

![[_attachments/qp-analysis/optimizer_planner.jpg]]

**해석**: `qo_planner_search()` 의 흐름:

```
qo_alloc_planner()      ← planner 객체 생성
qo_search_planner()     ← node 별 best plan 생성 (sarg term only)
for each partition:
   qo_search_partition_join()   ← first node 선별
   planner_permutate()          ← join 순서/method permutation
partition 병합
→ best plan
```

### Planner 자료구조

![[_attachments/qp-analysis/optimizer_planner2.jpg]]

**해석**: `node_info[N]` (N=node 갯수) 각각 `info { best_no_order: [plan|0|0|0], planvec: [plan|0|0|0|plan|0|0|0] (per eq_class), cardinality (ncard * selectivity(TC_sarg)) }`. `join_info[2^N]` 슬롯들은 bitmask 로 node 조합 표현:

```
bit:  111  110  101  100  011  010  001  000
slot: abc  bc   ac   c    ab   b    a    (empty)
```

가장 마지막 슬롯 (`111…1`) 이 `best_info` — 모든 node 조합의 최종 best plan 자리.

`planner->N/E/T/S/EQ/Q/P` = `QO_ENV` 의 `nnode/edge/nterm/nsegment/neqclass/subquery/partition`. `planner->M = 2^node` — 메모리 사용 ∝ 2^node (그래서 node 64 한계 + 실제로는 훨씬 작게 운영).

### Node 별 best plan (no_order)

![[_attachments/qp-analysis/optimizer_planner3.jpg]]

**해석**: query 의 TC_SARG term (`a.col2=1`, `b.col2=1`, `c.col2=1`) 만으로 각 node 의 독립 best scan plan 생성. 인덱스 있으면 인덱스 후보들끼리 plan_compare, 없으면 full scan. 결과는 `node_info[i].best_no_order`.

### Node 별 ordered planvec

![[_attachments/qp-analysis/optimizer_planner4.jpg]]

**해석**: join term (`a.col1=b.col1`, `b.col1=c.col1`) 으로 도출된 eq_class 마다, 해당 컬럼으로 정렬한 best plan 을 별도 슬롯에 저장 (`planvec[eq_class_idx]`). 이것은 **sort-merge join** 후보 plan 의 input 으로 사용.

### First node 선별 결정 트리

![[_attachments/qp-analysis/optimizer_planner5.jpg]]

**해석**: query `select count(1) from tbl1 a, tbl2 b where a.col1=b.col1, index(a.col1)`. node A (Tcard=50, Ncard=5000, cost=50) vs node B (100/10000/100).

```
                a,b cost compare
              ┌──── a < b ────┐    ┌──── a > b ────┐
              │                │    │                │
        comp card             ...  comp card        ...
       a<b      a>b           a<b      a>b
       f: a    f: a,b         f: a,b   a is index inner join?
                                       ┌── yes ──── f: b
                                       └── no  ──── f: a,b (← 이상한 케이스)
```

`f:` 가 first node 후보. `dep_set / outer_dep_set` node 는 자동 제외 (선행 node 필수), derived table 은 자동 포함 (인덱스 스캔 불가).

> 본문 author 의 지적: "first node 선정 시 cost 비교가 적절치 않다고 생각되며 card 를 기준으로 삼아야 할 것" + "index scan 가능 유무 확인이 오른쪽 분기에서만 수행" → **개선 후보 지점**.

### Permutation 실예

![[_attachments/qp-analysis/optimizer_planner6.jpg]]

**해석**: `select 1 from a,b,c where a.col1=b.col1 and b.col2=c.col2`. join term 존재: `(a,b)`, `(b,c)`. 가능한 순열: `A→B→C, A→C→B, B→A→C, B→C→A, C→A→B, C→B→A` (총 6).

실행 trace:
1. `A→B` plan cost 300 → `join_info[011]` 저장
2. `(A,B)→C` plan cost 500 → `join_info[111]` 저장
3. `A→C` — edge 없음, 진행 X
4. `B→A` plan cost 400 → `join_info[011]` 의 A→B (300) 와 비교, B→A 폐기. 후속 `B→A→C` 도 진행 X
5. `B→C` plan cost 600 → `join_info[111]` 의 500 보다 높음, 폐기 + 후속 진행 X
6. `C→A` — edge 없음
7. `C→B` plan cost 100 → `join_info[011]` (A→B 300) 보다 낮음 → 저장 후 진행
8. `(C,B)→A` plan cost 300 → `join_info[111]` (A→B→C 500) 보다 낮음 → 대체

→ best_info: `C→B→A` cost 300.

> permutation 완료 시 `best_info` 가 비어있으면 optimization 실패 → 기본 plan (휴리스틱) 으로 fallback.

## TO_DO (분석서 표기)

- join method 선택 로직 상세 (`nl-join` vs `idx-join` 구분 기준, sort merge join 진입 조건, `USE_MERGE` 힌트 처리)
- cost 산정 공식 (selectivity 곱 외 추가 가중치)
- plan compare 알고리즘 (어떤 metric 으로 두 plan 의 우열을 가리는가)
- plan 객체 메모리 재사용 패턴

→ [[components/optimizer]] 에 채워야 할 항목.

## Cross-references

- [[components/optimizer]] — 기존 component 페이지
- [[qp-analysis-rewriter]] — `qo_optimize_queries` (rewriter 부분) 후 본 단계로
- [[qp-analysis-xasl-generator]] — `qo_to_xasl(plan)` 으로 plan → XASL 변환
- 원문 한국어 본문: `.raw/qp-analysis-optimizer.md`
