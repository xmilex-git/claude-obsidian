---
type: concept
title: "QO_ENV"
status: stub
domain: cubrid-optimizer
aliases: ["query optimizer environment", "query graph"]
created: 2026-05-11
updated: 2026-05-11
tags:
  - concept
  - cubrid
  - optimizer
related:
  - "[[components/optimizer]]"
  - "[[sources/qp-analysis-optimizer]]"
---

# QO_ENV

CUBRID optimizer 가 PT_NODE 트리로부터 빌드하는 **query graph** 컨테이너. cost-based plan 탐색의 입력. NODE 갯수 **최대 64** (`qo_env_init` 의 절대 한계, `QO_PLANNER.M = 2^N` 메모리 비례).

## 핵심 어휘 (일반 RDBMS 대응)

| QO_ENV 용어 | 일반 RDBMS | 대응 구조 |
|---|---|---|
| **NODE** | table | `PT_SPEC`. `QO_NODE_NCARD` = row 수, `QO_NODE_TCARD` = page 수 |
| **SEGMENT** | column | 함수-기반 인덱스의 경우 expression |
| **TERM** | predicate | join 조건 + sarg 조건 통합 |
| **edge** | join term | term 중 join 인 것 |
| **PARTITION** | edge 없는 독립 node 그룹 | cartesian product 로 합쳐짐 |
| **EQCLASS** | join term 의 양변 segment 집합 | sort-merge join 의 sort key 후보 |

## TERM CLASS (bit-encoded)

`qo_analyze_term()` 에서 결정. 10가지 class — PATH/EDGE/FAKE 3-bit 인코딩. 대표: `QO_TC_SARG` (조회조건), `QO_TC_JOIN` (조인조건), `QO_TC_DURING_JOIN` (outer join on절), `QO_TC_AFTER_JOIN` (outer join where절), `QO_TC_DEP_LINK` (correlated subquery linkage), `QO_TC_DUMMY_JOIN` (보조).

자세히는: [[components/optimizer]] + [[sources/qp-analysis-optimizer]].

## 빌드 시퀀스

```
qo_optimize_query
  optimize_helper (사전 정보 생성)
    build_query_graph      / build_graph_for_entity   (qo_add_node + segment)
    qo_add_dep_term        (correlated subquery → QO_TC_DEP_LINK)
    qo_add_term            (PREDICATE term 추가)
    qo_add_dummy_join_term (sql-표준 outer 보조)
    qo_classify_outerjoin_terms
    qo_discover_edges      (edge sarg 앞으로 + selectivity sort)
    qo_assign_eq_classes   (sort-merge sort key)
    qo_discover_indexes    (index_entry → seg_equal_terms / seg_other_terms)
    qo_discover_partitions
  qo_planner_search (QO_PLANNER 로 plan 비교)
```

## See also

- [[concepts/PT_NODE]] — 입력
- [[concepts/XASL_NODE]] — 출력 (QO_PLAN → XASL_NODE)
