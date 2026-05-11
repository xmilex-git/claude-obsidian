---
type: component
parent_module: "[[modules/src|src]]"
path: "src/optimizer/"
status: active
purpose: "Cost-based query planning"
key_files: []
public_api: []
tags:
  - component
  - cubrid
  - optimizer
  - query
related:
  - "[[modules/src|src]]"
  - "[[Query Processing Pipeline]]"
  - "[[components/parser|parser]]"
  - "[[components/query|query]]"
  - "[[components/xasl|xasl]]"
  - "[[Build Modes (SERVER SA CS)]]"
created: 2026-04-23
updated: 2026-05-11
---

# `src/optimizer/` — Cost-Based Query Planner

Selects join orders, access paths, and physical operators based on cost estimates. Output feeds [[components/xasl|XASL]] generation in [[components/parser]].

## Side of the wire

> [!key-insight] Client-side
> Like the parser, the optimizer runs on the **client** (`#if !defined(SERVER_MODE)`). The server receives only the chosen plan as serialized XASL.

## Inputs / outputs

- **Input:** annotated `PT_NODE` tree from [[components/parser]] (after name resolution + semantic check)
- **Output:** decisions consumed by `xasl_generation.c` to build the `XASL_NODE` plan

## Selectivity defaults

`query_planner.c` defines a set of `DEFAULT_*_SELECTIVITY` constants used whenever no statistics-backed estimate is available. These live in the `.c` as file-private `#define`s:

| Constant | Value | Used for |
|---|---|---|
| `DEFAULT_NULL_SELECTIVITY` | 0.01 | `IS NULL` when null stats unavailable |
| `DEFAULT_EXISTS_SELECTIVITY` | 0.1 | `EXISTS (subq)` |
| `DEFAULT_SELECTIVITY` | 0.1 | generic fallback |
| `DEFAULT_EQUAL_SELECTIVITY` | 0.001 | `ATTR = const` |
| `DEFAULT_EQUIJOIN_SELECTIVITY` | 0.001 | `ATTR = ATTR` across relations |
| `DEFAULT_COMP_SELECTIVITY` | 0.1 | `ATTR {<,<=,>,>=} const` |
| `DEFAULT_BETWEEN_SELECTIVITY` | 0.01 | `ATTR BETWEEN a AND b` |
| `DEFAULT_IN_SELECTIVITY` | 0.01 | `ATTR IN (...)` |
| `DEFAULT_RANGE_SELECTIVITY` | 0.1 | composite range terms |

`PRM_ID_LIKE_TERM_SELECTIVITY` (system parameter) drives the default for `LIKE` / `LIKE ESCAPE`. These constants are the bedrock cost-model assumptions — they are calibration-sensitive and rarely touched.

`PRED_CLASS { PC_ATTR, PC_CONST, PC_HOST_VAR, PC_SUBQUERY, PC_SET, PC_OTHER, PC_MULTI_ATTR }` and the file-local `qo_classify` helper partition predicate operands for the selectivity dispatch.

## Inputs / outputs, continued

> [!update] 2026-05-11 — Two-stage internal architecture (per [[sources/qp-analysis-optimizer]])
> Optimizer 진입은 `parser_generate_xasl` 의 post-order walker (`parser_walk_tree`) 에서 `parser_generate_xasl_post` → `parser_generate_xasl_proc` → `pt_plan_query` → **`qo_optimize_query()`** (optimization) → `pt_to_buildlist_proc()` (XASL gen). PT_SELECT 만 진입, post-order 이므로 sub-query 가 먼저 plan/XASL 까지 완료된 뒤 상위에 연결.
>
> `qo_optimize_query()` 내부는 두 단계:
>
> 1. **사전 정보 생성** — `optimize_helper()` 가 `QO_ENV` 구조체 빌드
> 2. **Optimization 수행** — `qo_planner_search()` 가 `QO_PLANNER` 객체로 plan 비교 → best plan

## QO_ENV — CUBRID-특유의 query graph

원작 분석서가 정의하는 CUBRID 고유 어휘:

| 용어 | 일반 RDBMS 대응 | 비고 |
|---|---|---|
| **NODE** | table (PT_SPEC) | `QO_NODE_NCARD` = row 수, `QO_NODE_TCARD` = page 수 (통계) |
| **SEGMENT** | column | 함수 기반 인덱스의 경우 expression 그대로 (예: `seg1 = nvl(col1,1)`) |
| **final segment** | select list 의 컬럼 | bit set 으로 표현 |
| **TERM** | predicate | join term + 조회 term 통합 |
| **edge** | join term (term 중 join 인 것) | |
| **PARTITION** | edge 없는 독립 node 그룹 | partition 마다 따로 optimize 후 cartesian product 로 합침 |
| **EQCLASS** | join term 의 양변 segment 모음 | sort-merge join 의 sort key 후보 |

`qo_env_init()` 의 한계:
- **NODE 갯수 64 초과 시 에러** (cubrid 의 절대 한계, `QO_PLANNER.M = 2^node` 메모리 비례).
- `qo_validate()` 가 parse tree 로부터 node/term/segment 갯수 예측 후 `Nnodes/Nsegs/Nterms/Neqclasses` (할당 max) 와 `nnodes/nsegs/nterms/neqclasses` (실제 갯수) 로 양분 관리.

## TERM CLASS — bit-encoded

```
TERM CLASS               VALUE  PATH  EDGE  FAKE  NUM
QO_TC_PATH               0x30    1     1     0    000   ORDBMS PATH join
QO_TC_JOIN               0x11    0     1     0    001   일반 join predicate
QO_TC_DEP_LINK           0x1c    0     1     1    100   correlated subquery linkage
QO_TC_DEP_JOIN           0x1d    0     1     1    101
QO_TC_DUMMY_JOIN         0x1f    0     1     1    111   sql-표준 outer join 보조
QO_TC_SARG               0x02    0     0     0    010   일반 조회조건
QO_TC_OTHER              0x03    0     0     0    011
QO_TC_DURING_JOIN        0x04    0     0     0    100   outer join on 절
QO_TC_AFTER_JOIN         0x05    0     0     0    101   outer join where 절
QO_TC_TOTALLY_AFTER_JOIN 0x06    0     0     0    110
```

3개 비트 자리: **PATH** (indirect relation, ORDBMS 컬렉션), **EDGE** (join 관련), **FAKE** (실제 predicate 없는 가상 term).

- `qo_analyze_term()` 에서 모든 term 의 class + selectivity + indexable 평가 (op/LHS/RHS type 만으로 1차 판정; 실제 인덱스 매칭은 `qo_discover_indexes()` 와 `qo_generate_join_index_scan()`).
- `qo_classify_outerjoin_terms()` 가 추가로 DURING/AFTER 구분 + `node.outer_dep_set` (permutation 제약) 설정.

## QO_PLAN — plan_type 별 union

| `plan_type` | `plan_un` 활성 멤버 |
|---|---|
| `PLANTYPE_SCAN` | `scan_method` (seq/idx), term (key range), kf_term (key filter), sarg term (data filter), `qo_node` |
| `PLANTYPE_JOIN` | `outer`, `inner`, `join_method` (nl-join / idx-join / sort-merge) |
| `PLANTYPE_SORT` | `sort_type` (ORDERBY/...), `subplan` |

PT_NODE 의 `info` union 패턴과 동형 — `plan_type` 이 `plan_un` 의 활성 멤버를 결정.

## QO_PLANNER — node_info × join_info

- `node_info[N]` — 각 node 의 best_no_order (정렬 없는 best) + `planvec` (eq_class 별 ordered plan; sort-merge 후보).
- `join_info[2^N]` — bitmask-indexed slot. `bit 111…1` = `best_info` (최종 best plan 자리).

예 (N=3): `111=abc=best_info, 110=bc, 101=ac, 100=c, 011=ab, 010=b, 001=a`.

`planner->N/E/T/S/EQ/Q/P` = `QO_ENV` 의 `nnode/edge/nterm/nsegment/neqclass/subquery/partition`. `planner->M = 2^node` — 메모리 ∝ 2^node (=> 64 노드 한계 + 실 운영은 훨씬 작게).

## qo_planner_search() 흐름

```
qo_alloc_planner()              ← planner 객체 생성
qo_search_planner()             ← node 별 best plan (sarg term only)
for each partition:
   qo_search_partition_join()   ← first node 선별
   planner_permutate()          ← join 순서/method permutation
partition 병합
→ best plan
```

### Sequential pre-info 생성 (`optimize_helper()`)

1. `build_query_graph()` + `build_graph_for_entity()` — node 추가 (`qo_add_node`), segment 추가 (`info.spec.referenced_attrs`)
2. **DEPENDENT TABLE 처리** (`PT_IS_SET_EXPR`, `PT_IS_CSELECT`, `PT_DERIVED_JSON_TABLE`, `PT_IS_SUBQUERY && correlation_level==1`) — `qo_expr_segs()`/`qo_seg_nodes()`/`qo_add_dep_term()` 가 `subquery.dep_set` 채우고 `QO_TC_DEP_LINK` FAKE term 생성
3. **ON_COND term + WHERE PREDICATE term 추가** (`qo_add_term`/`qo_analyze_term`)
4. **DUMMY JOIN TERM** — `qo_add_dummy_join_term()`: sql-표준 outer join 의 연속 node 간 직접 join term 없을 때 보조
5. **`qo_classify_outerjoin_terms()`** — DURING/AFTER class 구분 + outer_dep_set
6. `qo_add_final_segment()` — select_list/group by/having/connect by segment 포함
7. **`qo_discover_edges()`** — edge 를 sarg 앞으로 정렬 + selectivity 내림차순. node selectivity = node_card × sarg_selectivity 곱
8. `qo_assign_eq_classes()` — join term 양변 segment 를 eq class 로 (sort-merge join 의 sort key 후보)
9. **`qo_discover_indexes()`** — `index_entry->seg_equal_terms[idx]` (op `=` 또는 RANGE LIST 한번) vs `seg_other_terms[idx]` (그 외)
10. `qo_discover_partitions()` — edge 없는 node 그룹 분리

### Optimization 본 단계

1. **planner 객체 생성** (`qo_alloc_planner`).
2. **Node 별 best_no_order plan** — TC_SARG term 만으로 single-node best. 인덱스 있으면 각 인덱스 candidate plan 끼리 `plan_compare`, 없으면 full scan.
3. **Node 별 ordered planvec** — 각 eq_class 의 컬럼으로 정렬한 best plan (sort-merge 후보 input).
4. **first node 선별** — `dep_set/outer_dep_set` node 자동 제외, derived table 자동 포함. 나머지 node 들끼리 cost compare → card compare → "is index inner join?" 결정 트리.
5. **permutation** — 가능한 join 순서마다 plan 생성 후 `join_info[bitmask]` 의 기존 plan 과 비교. join term 없는 페어 (edge 없는) 는 진행 X. cost 비교 후 worse plan + 후속 순열 모두 폐기.

> [!warning] First-node 선별 비효율 — 원작자 본인 지적
> 분석서 author 가 직접 지적: "first node 선정시 cost 의 비교 로직이 적절치 않다고 생각되며 card (예측되는 row 수) 를 기준으로 삼아야 할 것 … index scan 가능 유무에 대한 확인을 오른쪽 분기에서만 수행하기 때문에 발생". 비효율 시나리오: node b 에 인덱스 없는데 cost 만으로 a → b 가 선택되어 b → a 보다 못한 plan 이 출력될 수 있음.

## TO_DO (분석서 표기)

- **join method 선택 로직** — `nl-join` vs `idx-join` 구분 기준, sort-merge join 진입 조건, `USE_MERGE` 힌트 처리 (현 기본값: sort-merge 는 `USE_MERGE` 힌트 시에만)
- **cost 산정 공식** — selectivity 곱 외 가중치
- **plan compare 알고리즘**
- **plan 객체 메모리 재사용 패턴**

## Cross-cutting insight — 3-way predicate split

`qo_analyze_term()` 에서 결정되는 [`WHERE_RANGE / WHERE_KEY / WHERE_PRED`](sources/qp-analysis-xasl-generator.md#WHERE_RANGE-WHERE_KEY-WHERE_PRED) 의 3-분할은 인덱스 정의 + predicate 의 컬럼 매칭으로 정해지며 XASL generator 에서 `regu_list_range/key/pred` 로 분배된다. 즉 optimizer 의 term-class 평가가 XASL 의 인덱스 스캔 효율을 직접 결정.

## From the Manual (sql/tuning.rst, sql/parallel.rst — added 2026-04-27)

> [!gap] Documented contracts
> - **Full SQL HINT catalogue (28 hints)** is documented in `sql/tuning.rst`. See [[sources/cubrid-manual-sql-tuning-parallel]] for the table. Notable additions in 11.4: `USE_HASH`/`NO_USE_HASH` (HASH JOIN opt-in), `LEADING(t1 t2)` (finer than ORDERED).
> - **`SET OPTIMIZATION LEVEL n`** valid values: 1 (full, default), 2 (heuristic only), +256 (emit plan trace), +512 (emit join enumeration trace). So 1, 2, 257, 258, 513, 514.
> - **Plan cache regeneration** triggered ONLY when **BOTH**: (a) ≥6 minutes elapsed since last check, AND (b) `UPDATE STATISTICS` changed page count by ≥10× since plan was cached. (`sql/tuning.rst:14-20`).
> - **NEW 11.4 stat improvements**: sampling pages 1000 → **5000**; NDV (number of distinct values) collection improved; NOT LIKE selectivity added; function-based-index selectivity added; NDV duplicate weighting (>1% sample dup → reweight); `SSCAN_DEFAULT_CARD` to prevent inefficient NL JOIN on tiny cardinality estimates; LIMIT cost/cardinality reflected in plans.
> - **Index Skip Scan auto-selected in 11.4** — no `INDEX_SS` hint needed.
> - **Optimizer prefers cheaper index over PK** when applicable (NEW 11.4).
> - **Stored Procedure execution plans** now use index scans, eliminate unnecessary joins, and support result caching in correlated subqueries.

See [[sources/cubrid-manual-sql-tuning-parallel]] for the full hint catalogue and 11.4 optimizer changes.

## Related

- Parent: [[modules/src|src]]
- [[Query Processing Pipeline]]
- Source: [[cubrid-AGENTS]]
- Manual: [[sources/cubrid-manual-sql-tuning-parallel]]
- Source (internal R&D wiki): [[sources/qp-analysis-optimizer]] — 9 architecture diagrams + step-by-step
- Related rewriter: [[components/optimizer-rewriter]] — runs first; see [[sources/qp-analysis-rewriter]]
