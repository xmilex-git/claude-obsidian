---
type: source
source_type: jira-wiki
title: "QP Analysis — Rewriter"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-rewriter"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - rewriter
  - query-transformation
status: active
note: "원본에 TO_DO 마커 다수 (작성 미완)"
related:
  - "[[qp-analysis]]"
  - "[[components/optimizer-rewriter]]"
  - "[[components/optimizer-rewriter-select]]"
  - "[[components/optimizer-rewriter-set]]"
  - "[[components/optimizer-rewriter-subquery]]"
  - "[[components/optimizer-rewriter-term]]"
  - "[[components/optimizer-rewriter-unused-function]]"
  - "[[components/optimizer-rewriter-auto-parameterize]]"
attachments:
  - "_attachments/qp-analysis/rewriter.jpg"
---

# QP Analysis — Rewriter (Query Transformation)

> 원본: [analysis-for-qp-rewriter](http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-rewriter). **분석서 본문에 TO_DO 마커 다수 — 원작자도 미완성으로 표기**.

## 진입 흐름

`mq_translate()` 가 시작점. 큰 흐름:

```
mq_translate
└── mq_translate_helper
    └── mq_push_paths
        └── mq_check_rewrite_select
            (where) pt_cnf → pt_transform_cnf_pre / pt_transform_cnf_post
            (from)  mq_is_union_translation
                    mq_rewrite_vclass_spec_as_derived
                    mq_copypush_sargable_terms
            mq_push_paths_select
└── mq_translate_local
└── mq_bump_order_dep_corr_lvl
└── parser_walk_tree(parser, node, mq_mark_location, NULL,
                     mq_check_non_updatable_vclass_oid, &strict)
└── mq_optimize
    └── qo_optimize_queries   ← 재작성 본체
```

`qo_optimize_queries` 안에서 호출되는 재작성 함수들 (분석서 직인용 호출 트리, 위→아래 순서):

```
qo_optimize_queries
  pt_split_join_preds
  qo_can_generate_single_table_connect_by
  qo_move_on_clause_of_explicit_join_to_where_clause
  qo_rewrite_index_hints
  qo_analyze_path_join_pre, qo_analyze_path_join
  qo_rewrite_subqueries
  pt_cnf
  qo_reduce_equality_terms
  qo_converse_sarg_terms
  qo_reduce_comp_pair_terms
  qo_rewrite_like_terms
  qo_convert_to_range
  qo_apply_range_intersection
  qo_fold_is_and_not_null
  qo_rewrite_outerjoin
  qo_rewrite_innerjoin
  qo_rewrite_oid_equality
  qo_reduce_order_by
  qo_do_auto_parameterize
  pt_rewrite_to_auto_param
  qo_do_auto_parameterize_limit_clause
  qo_do_auto_parameterize_keylimit_clause
  qo_rewrite_hidden_col_as_derived
```

## CNF 변환의 시각화

![[_attachments/qp-analysis/rewriter.jpg]]

**해석**: `col1=1 AND col2=1` 의 변환 전/후 비교.

- **변환 전 (PT_NODE)**: `PT_EXPR Op=PT_AND` 가 root, Arg1/Arg2 로 두 자식 `PT_EXPR PT_EQ` 보유. 트리 형태로 표현된 AND.
- **변환 후 (Optimized PT_NODE)**: AND/OR 가 더 이상 별도 노드가 아니라 **포인터 연결**. 같은 레벨의 `PT_EXPR PT_EQ` 가 `next` (AND-chain) 와 `or_next` (OR-chain) 로 연결됨.

> 이후 predicate 순회는 본문이 직접 정의한 패턴:
> ```
> while PT_NODE.next
>    predicate 처리
>    while PT_NODE.or_next
>      predicate 처리
> ```

→ `PT_AND` / `PT_OR` 가 살아있는 predicate 는 **CNF 변환 실패** 의미. CNF 변환이 안 된 predicate 는 인덱스 스캔 불가. CUBRID 는 100항 초과 시 변환 포기.

## 주요 재작성 카탈로그

### `pt_split_join_preds` — Hierarchical query 의 JOIN vs SEARCH 분리

```sql
FROM TBL1 A, TBL2 B
WHERE A.COL1 = B.COL1                          -- JOIN PREDICATE
      AND B.COL2 = 3                           -- SEARCH PREDICATE
      AND B.COL3+A.COL3 = serial.curent_value  -- SEARCH PREDICATE (subquery/serial 포함)
CONNECT BY PRIOR A.COL4 = A.COL3
```

JOIN PREDICATE 는 `where` 에, SEARCH PREDICATE 는 `after_cb_filter` 에 분리 저장 (CONNECT BY 실행 순서: JOIN → CONNECT BY → SEARCH).
구분 기준 (`pt_must_be_filtering`): (1) 두 개 이상 spec_id 포함 안 함, (2) 두 개 이상 spec_id 라도 subquery/serial/rownum 포함 시 SEARCH.

### `qo_move_on_clause_of_explicit_join_to_where_clause`

ANSI `inner join … on` → conventional `, … where` 로 임시 이동. `qo_optimize_queries_post()` 에서 복구.

```sql
table a inner join table b on a.col1 = b.col1
   ==>  table a, table b where a.col1 = b.col1
```

이유: 이후 일부 최적화가 conventional 표현에서 더 단순.

### `qo_reduce_equality_terms`

```sql
col1 = 1 and col2 <> col1                ==>  col1 = 1 and col2 <> 1
from tbl a, (select 1 col1 from tbl) b where a.col1 = b.col1
   ==> ... where a.col1 = 1
```

`=` 로 묶인 segment 가 상수일 때 다른 predicate 의 같은 segment 를 상수로 치환.

### `qo_rewrite_outerjoin` — Outer → Inner 변환

조건: outer join 의 dangling-side node 의 predicate 가 NULL 을 허용하지 않음 → `PT_JOIN_INNER` 로 변경. `qo_check_nullable_expr()` 가 nullable op 판정.

```sql
-- 변환 OK
table a left outer join table b on a.col1 = b.col1 where b.col2 = 3
   ==> table a inner join table b on a.col1 = b.col1 where b.col2 = 3

-- 변환 X (nvl 이 null 허용)
table a left outer join table b on a.col1 = b.col1 where nvl(b.col2,1) = 3
```

### `qo_rewrite_innerjoin` — explicit → implicit

SQL 표준 `inner join … on` → `,` + WHERE.

```sql
table a inner join table b on a.col1 = b.col1
   ==> table a, table b where a.col1 = b.col1
```

제약: explicit inner 가 outer 와 인접하면 변환 불가 — 분석서 본문에 author 의 의문 표기. 현재 동작은 SQL 표준 여부에 의해 분기. 의도된 동작인지 확인 필요로 남김.

### `pt_rewrite_to_auto_param` — host variable 치환

상수 → `?` 로 치환하여 plan cache hit rate 향상. `cubrid.conf` `HOSTVAR_LATE_BINDING=yes` 로 비활성화.

대상 op: `PT_EQ, PT_GT, PT_GE, PT_LT, PT_LE, PT_LIKE, PT_ASSIGN, PT_BETWEEN, PT_RANGE`.

```sql
-- 변환 OK
where col1 = 1 and col2 in (1,2,3)
   ==> col1 = ? and col2 range ( ?= or ?= or ?= )

-- 변환 X (set type RHS)
col1 = 1 and col2 in ((1,1),(2,2))
   ==> col1 = ? and col2 range ((1,1)= or (2,2)= )
-- 10.2 에서 반영 예정 / 10.1 에서 set 을 단일 host var 로 변환하는 오류는 수정됨
```

`in` 은 `qo_convert_to_range()` 에서 range op 형태로 변환됨.

## TO_DO (분석서 표기)

- REWRITE 성질에 따른 분류 작성 미완 (VIEW / PREDICATE / JOIN / …)
- 위 카탈로그 외 함수들 (`qo_apply_range_intersection`, `qo_fold_is_and_not_null`, `qo_reduce_order_by`, `qo_rewrite_oid_equality`, `qo_rewrite_subqueries`, view/CTE rewrite path 등) 의 로직 요약 미작성

→ 향후 보강 시 [[components/optimizer-rewriter]] 와 그 sub-component 들 (subquery / term / set / select / unused-function / auto-parameterize) 에 채워 넣어야 함.

## Cross-references

- [[components/optimizer-rewriter]] + sub-component pages
- [[qp-analysis-semantic-check]] — `mq_translate` 직전 단계
- [[qp-analysis-optimizer]] — `qo_optimize_queries` 다음 단계 (cost-based plan 탐색)
- 원문 한국어 본문 보존: `.raw/qp-analysis-rewriter.md`
