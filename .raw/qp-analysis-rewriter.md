---
source: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-rewriter"
source_type: jira-wiki
title: "analysis for QP rewriter"
slug: qp-rewriter
fetched_at: 2026-05-11
captured_via: playwright
language: ko
domain: cubrid-query-processing
attachments: []
status_note: "Sub-page contains TO_DO markers — author marks this as incomplete."
---

# analysis for QP rewriter

overview : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-

## REWRITER 란?

PARSER에서 사용자가 작성한 query를 그대로 parse tree로 변환했다면, REWRITER에서는 parse tree의 구조를 변경하여 최적화 작업을 진행합니다.
간단하게 말하면 query를 재작성하는 것이며, 성능을 위한 부분과 이후 OPTIMIZATION을 진행하기 위한 준비하는 부분이 있습니다.
query를 재작성하는 것이 꼭 REWRITER 단계에서만 진행되는 것은 아니고, PARSER와 SEMANTIC CHECK 단계에서도 진행됩니다.
다만 REWRITER 단계에 query를 재작성하는 로직이 가장 많이 포함되어 있습니다. 이후 OPTIMIZER에서의 로직을 이해하기 위해서도 사전에 어떻게 쿼리가 재작성되는지 확인이 필요합니다.

TO_DO) REWRITE 성질에 따른 구분 작성 필요.
1. VIEW에대해 처리하는 부분
2. PREDICATE 관련 부분
3. JOIN 관련 부분
등등

## 함수 콜 트리

`mq_translate()` 함수가 시작 지점이며 아래는 관련 주요 함수들입니다.

```
mq_translate
mq_translate_helper
mq_push_paths
 mq_check_rewrite_select
   (where)
   pt_cnf
    pt_transform_cnf_pre
    pt_transform_cnf_post
   (from)
   mq_is_union_translation
   mq_rewrite_vclass_spec_as_derived
   mq_copypush_sargable_terms
 mq_push_paths_select
mq_translate_local
mq_bump_order_dep_corr_lvl
parser_walk_tree (parser, node, mq_mark_location, NULL, mq_check_non_updatable_vclass_oid, &strict);

mq_optimize
qo_optimize_queries
*  (rewrite)
  pt_split_join_preds
  qo_can_generate_single_table_connect_by
  qo_move_on_clause_of_explicit_join_to_where_clause
  qo_rewrite_index_hints
  qo_analyze_path_join_pre, qo_analyze_path_join
  qo_rewrite_subqueries
  pt_cnf
  qo_reduce_equality_terms   :  segment가 상수 값일 경우 다른 predicate의 해당 segment도 치환한다.
  qo_converse_sarg_terms
  qo_reduce_comp_pair_terms
  qo_rewrite_like_terms
  qo_convert_to_range
  qo_apply_range_intersection
  qo_fold_is_and_not_null
  qo_rewrite_outerjoin : outerjoin으로 작성되었으나 inner join으로 변경 가능할 경우 join type 변경( null이 불가한 경우)
  qo_rewrite_innerjoin : explicit inner join을 implicit inner join으로 변환
  qo_rewrite_oid_equality
  qo_reduce_order_by
  (host val)
  qo_do_auto_parameterize
  pt_rewrite_to_auto_param : 상수 변수를 host variable로 변환함. plan cache의 hit ratio 증가를 위함.
  qo_do_auto_parameterize_limit_clause,  qo_do_auto_parameterize_keylimit_clause
  qo_rewrite_hidden_col_as_derived
```

아래부터는 각 함수들이 어떤 재작성을 하는지에 대한 설명입니다.

### pt_split_join_preds

계층형 쿼리(Hierarchical query)는 `JOIN PREDICATE → CONNECT BY → SEARCH PREDICATE` 순으로 수행된다.
위 순서로 수행하기 위해 JOIN predicate, SEARCH predicate를 구분한다.

- `node->info.query.q.select.where` 에는 JOIN PREDICATE만 남기고
- `node->info.query.q.select.after_cb_filter` 에 SEARCH PREDICATE를 저장한다.

구분하는 기준은 아래와 같다. (`pt_must_be_filtering`)
1. predicate에서 두개 이상의 `spec_id` (테이블 ID) 를 갖지 않는 경우 search predicate
2. 두개 이상의 spec_id를 포함해도 subquery, serial, rownum이 포함되면 search predicate

```sql
FROM TBL1 A, TBL2 B
WHERE A.COL1 = B.COL1                          -- JOIN PREDICATE
      AND B.COL2 = 3                           -- SEARCH PREDICATE
      AND B.COL3+A.COL3 = serial.curent_value  -- SEARCH PREDICATE
CONNECT BY PRIOR A.COL4 = A.COL3
```

### qo_move_on_clause_of_explicit_join_to_where_clause

explicit join (SQL 표준) 방식으로 작성한 `on` 조건절을 `where` 절로 이동한다. `qo_optimize_queries_post()` 에서 다시 복구되며, 몇가지 최적화를 위해 임시로 이동한다.

- `spec->info.spec.on_cond` 에서 `where->next` 로 이동

```sql
table a inner join table b on a.col1 = b.col1
==> table a, table b where a.col1 = b.col1
```

### qo_reduce_equality_terms

`=` 오퍼레이터를 통해 특정 segment가 상수일 경우 다른 predicate에 해당 값을 상수로 치환한다.

```sql
col1 = 1 and col2 <> col1  ==>  col1 = 1 and col2 <> 1
from tbl a, (select 1 col1 from tbl) b where a.col1 = b.col1
   ==>  ... where a.col1 = 1
```

### qo_rewrite_outerjoin

outer join으로 작성되었으나, inner join으로 변환 가능할 경우 `join_type`을 `PT_JOIN_INNER` 으로 변경한다.

조건: join 조건 이외에 해당 node의 predicate가 존재하며 해당 predicate가 null이 조회 불가한 경우.
`qo_check_nullable_expr()` : null이 가능한 OP 체크하는 함수.

```sql
-- 1. 변환되는 케이스
table a left outer join table b on a.col1 = b.col1 where b.col2 = 3
   ==> table a inner join table b on a.col1 = b.col1 where b.col2 = 3
-- 2. 변환되지 않는 케이스
table a left outer join table b on a.col1 = b.col1 where nvl(b.col2,1) = 3
```

### qo_rewrite_innerjoin

explicit inner join을 implicit inner join으로 변환. SQL 표준으로 작성된 join을 where절 조인조건을 사용하는 쿼리로 재작성한다.

- explicit join : SQL 표준 join 작성 (예: `tbl a inner join tbl b on a.col1 = b.col1`)
- implicit join : join 조건이 where 절의 predicate로 작성됨 (예: `tbl a, tbl b where a.col1 = b.col1`)

implicit inner join으로 변환 불가능한 조건은 아래와 같다.
조건: SQL 표준으로 작성된 inner join이 outer join과 연결되어 있는 경우에 변경이 불가함.

- `a inner join b inner join c outer join d` ==> 변환 불가
- `a inner join b, c outer join d where ...` ==> `a, b, c outer join d where ...` 변환됨.

```sql
-- 1. 변환되는 케이스
table a inner join table b on a.col1 = b.col1
   ==> table a, table b where a.col1 = b.col1
-- 2. 변환되지 않는 케이스에 대한 의문점
-- 1. a inner b left outer c
-- 2. a left outer b inner c
-- 1번은 변경 가능해 보이고 2번은 순서가 정해져 변경하면 안 된다고 생각됨
-- 하지만 1,2 모두 변경 안됨.
-- 현재 조건이 SQL 표준 작성이냐 아니냐에 따라 결과가 달라지는데 의도한 사항인지 확인이 필요함.
-- <<지금 생각으로는 2번처럼 outer b node에 inner join 되는 c node만 제한해야될 듯함.>>
```

### pt_rewrite_to_auto_param

CUBRID는 기본적으로 쿼리의 상수 변수를 host variable로 변환한다. 상수의 값이 다른 경우 다른 쿼리로 인식해 plan cache를 이용하지 못함을 막기위해서다.
`cubrid.conf` 에서 `HOSTVAR_LATE_BINDING=yes` 로 설정시 해당 기능을 사용하지 않을 수 있다.

변환의 대상이 되는 operation:
```
PT_EQ, PT_GT, PT_GE, PT_LT, PT_LE, PT_LIKE, PT_ASSIGN, PT_BETWEEN, PT_RANGE
```

```sql
-- 1. 변환이 되는 케이스
where col1 = 1 and col2 in  (1,2,3)
   ==> col1 = ? and col2 range ( ?= or ?= or ?=)
-- in OP는 qo_convert_to_range() 에서 range op를 사용하는 형식으로 변경된다.
-- 2. 변환이 되지 않는 케이스
col1 = 1 and col2 in ((1,1),(2,2))
   ==> col1 = ? and col2 range ((1,1)= or (2,2)=)
-- range OP에 대해서 RHS가 set type일 경우 변환되지 않음 (10.2에서 반영 예정).
-- 10.1 기준 변경이 되나, set type이 하나의 host var로 변경되는 것은 오류이며 수정되었다.
```

TO_DO) 작성되지 않은 주요함수들의 로직 요약 필요
