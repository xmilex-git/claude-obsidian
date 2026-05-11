---
type: source
source_type: jira-wiki+pdf
title: "QP Analysis — Semantic Check"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-resolve-names"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - semantic-check
  - name-resolution
  - type-checking
  - constant-folding
status: active
related:
  - "[[qp-analysis]]"
  - "[[components/semantic-check]]"
  - "[[components/parser]]"
pdf_sources:
  - ".raw/qp-pdfs/semantic_check_overview_v1_0.pdf (3p)"
  - ".raw/qp-pdfs/semantic_check_name_resolution_v0_9.pdf (22p)"
  - ".raw/qp-pdfs/semantic_check_type_checking_constant_folding_v1_0.pdf (5p)"
  - ".raw/qp-pdfs/semantic_check_particular_statement_v0_8.pdf (6p)"
---

# QP Analysis — Semantic Check

> 원본 wiki (slug `resolve-names`, 제목 `analysis-for-qp-semantic-check`) 본문은 없고 PDF 4개로 구성. 본 페이지는 4개 PDF 종합 + cheat sheet 참조.

## 4단계 구성 (overview PDF + cheat sheet)

```
pt_compile()
└── pt_semantic_check()
    └── pt_check_with_info()
        switch (node->node_type) case PT_SELECT:
            1) pt_resolve_names()              // name resolution
            2) pt_check_where()                // aggregate/analytic in WHERE 금지
            3) pt_check_and_replace_hostvar()  // ? → PT_VALUE
            4) pt_semantic_check_local()       // semantics + type-check + constant-fold
            5) parser_walk_tree(pt_expand_isnull_preds)  // path expression
```

세션 단위로 statement 별 진입. 각 PT_SELECT 마다 위 5단계 + path expansion 수행.

## 1. Name Resolution — `pt_resolve_names()`

PT_NAME 의미 (테이블명/별명/컬럼명) 분석 + `PT_SPEC` 과 binding.

### 5-step 함수 호출 (cheat sheet)

```c
pt_resolve_names()
  /* (a) entity spec 을 flat list 로 */
  parser_walk_tree(statement, pt_flat_spec_pre, ...);
  /* (b) name binding */
  parser_walk_tree(statement, pt_bind_names, &bind_arg, pt_bind_names_post, &bind_arg);
  /* (c) group by/having alias 연결 */
  parser_walk_tree(statement, pt_resolve_group_having_alias, ...);
  /* (d) natural join → inner/outer join 변환 */
  parser_walk_tree(statement, NULL, NULL, pt_resolve_natural_join, NULL);
  /* (e) SELECT ... FOR UPDATE 처리 */
  if (PT_SELECT_INFO_IS_FLAGED(stmt, PT_SELECT_INFO_FOR_UPDATE)) { ... }
```

### SPEC 정보 생성 — `pt_flat_spec_pre()`

PT_SPEC 노드를 발견하면 3가지 정보 저장:

1. **`id` / `spec_id`** — `PT_SPEC.id = (UINTPTR)node` (자기 주소). 각 `PT_NAME.spec_id` 는 binding 결과로 이 값을 가짐 → 어느 PT_SPEC 에 속한 컬럼인지 추적.
2. **`db_object`** — pre-fetch 단계가 workspace 에 cache 해둔 `DB_OBJECT` 를 연결.
3. **`flat_entity_list`** — entity_name 이 가리키는 테이블 + (partition / 상속) 관계로 함께 가져와야 할 테이블들의 `PT_NAME` 리스트.

`flat_entity_list` 의 존재 이유: ORDBMS 의 `ALL ... EXCEPT` 구문 + partition table 처리 (예: `FROM ALL super_class EXCEPT sub_class` → 상위 + 형제 partition 들의 정보를 한꺼번에 매달아야 함). 단순 SELECT 에서는 entity_name 과 거의 동일.

### SPEC 정보 연결 — `pt_bind_names()`

분석서 본문 (간단한 예시 `SELECT s_name FROM code`):

- name binding 은 **scope stack** (`PT_BIND_NAMES_ARG` + `SCOPES` linked list) 으로 관리. 구문 시작 노드 (`PT_SELECT/PT_UPDATE/...`) 진입 시 새 scope 가 push, 더 좁은 scope 가 스택 top.
- `pt_bind_name_or_path_in_scope()` 에서 stack 내 specs 를 순회하며 컬럼명 매칭. 매치 시:
  - `spec_id` 에 PT_SPEC 의 id 입력.
  - `resolved` 에 테이블명/alias 입력.
  - PT_SPEC 의 DB_OBJECT 에서 컬럼 정보 가져와 `PT_DATA_TYPE` 노드 생성 + `DB_DOMAIN` 연결.

### `*` resolution — `pt_resolve_star()`

`SELECT *` 시 `PT_VALUE(PT_TYPE_STAR)` 가 들어있던 자리에 PT_SPEC 의 모든 컬럼명을 `PT_NAME` 노드 리스트로 펼침. `<table>.*` 도 동일 메커니즘.

### Derived table & subquery name binding

```sql
SELECT SUM(n), AVG(n)
FROM (SELECT gold FROM participant WHERE nation_code = 'KOR') AS t(n);
```

1. 제일 바깥 PT_SELECT 진입 시 `pt_bind_scope` 가 **derived table 부터 우선 처리**.
2. 안쪽 SELECT 의 name binding 완료.
3. `pt_semantic_type()` 으로 안쪽 노드들 타입 지정.
4. derived table 의 select list 와 상위 PT_SPEC 의 `as_attr_list` 를 매칭 (`(n)` 이 없으면 select list 컬럼명 그대로 as_attr_list 새로 생성). 갯수 일치 필수.

### Join syntax — `PT_JOIN_TYPE`

```c
typedef enum {
  PT_JOIN_NONE        = 0x00,
  PT_JOIN_CROSS       = 0x01,
  PT_JOIN_NATURAL     = 0x02,  /* not used (NATURAL은 spec.natural=true flag로 표현) */
  PT_JOIN_INNER       = 0x04,
  PT_JOIN_LEFT_OUTER  = 0x08,
  PT_JOIN_RIGHT_OUTER = 0x10,
  PT_JOIN_FULL_OUTER  = 0x20,  /* not used (정의만; 미구현) */
  PT_JOIN_UNION       = 0x40,  /* not used */
} PT_JOIN_TYPE;
```

ANSI 표준 `A INNER JOIN B ON …` 의 경우 join 조건이 B 의 `on_cond` 에 매달리고 A 는 `PT_JOIN_NONE`. Conventional `FROM A, B WHERE A.x = B.x` 는 join 조건이 WHERE 에 들어가고 둘 다 `PT_JOIN_NONE`. PT_SPEC `next` 로 연결되며 `location` 이 0,1,2,… 순서로 증가. on_cond 하위 노드 모두 동일 location.

### Oracle-style outer join 변환

`single_column(+) op expression` (right outer) / `expression op single_column(+)` (left outer) 패턴.

1. 파서가 `PT_EXPR_INFO_LEFT_OUTER` / `PT_EXPR_INFO_RIGHT_OUTER` flag 를 PT_EXPR 에 설정.
2. SELECT 노드에 `PT_SELECT_INFO_ORACLE_OUTER` flag 가 set 되어 있으면 → 2-pass 검색으로 `(+)` 가진 predicate 의 lhs/rhs spec 의 location 비교 후 ANSI outer join 으로 재배치.
3. `WHERE` 절 → `on_cond` 이동, 적절한 `PT_JOIN_LEFT_OUTER` / `PT_JOIN_RIGHT_OUTER` 지정.

### Natural join resolution — `pt_resolve_natural_join()`

`A NATURAL JOIN B` (spec.natural=true) 발견 시:

- A 의 attribute list (entity → DB_OBJECT → DB_ATTRIBUTE, 또는 derived/CTE 의 경우 select list/with 절) 와 B 의 attribute list 비교.
- 이름이 같은 컬럼마다 `A.col = B.col` PT_EXPR 노드 생성 → B 의 `on_cond` 에 추가. 결과적으로 INNER JOIN 으로 재작성.

### FOR UPDATE flag

`SELECT … FOR UPDATE [OF spec_list]` 는 X_LOCK 대상을 결정. 이 단계에서는 단순히 `PT_SELECT_INFO_FOR_UPDATE` flag 만 set.

## 2. WHERE 절의 aggregate/analytic 금지 — `pt_check_where()`

```sql
-- error
SELECT … WHERE SUM(a.cost) > p.budget AND p.id IS NOT NULL
```

WHERE 절 재귀 순회 → aggregate 또는 analytic function 발견 시 즉시 에러.

- `MSGCAT_SEMANTIC_INVALID_AGGREGATE`
- `MSGCAT_SEMANTIC_NESTED_ANALYTIC_FUNCTIONS`

## 3. Host variable → PT_VALUE — `pt_check_and_replace_hostvar()`

`?` (PT_HOST_VAR) 노드를 PT_VALUE 타입으로 변환. prepared statement bind 단계에서 실제 값이 들어옴.

## 4. Statement-별 의미 체크 — `pt_semantic_check_local()`

PT_SELECT 진입 시 수행 (요약):

| 검사 | 의도 |
|---|---|
| `pt_length_of_select_list()` | illegal multi-column subquery (parser 에서 이미 잡지만 한번 더) |
| `pt_check_into_clause()` | INTO 컬럼 갯수 = `:identifier` 갯수, subquery 내 INTO 금지 |
| **WITH INCREMENT** | `... WITH INCREMENT FOR read_count` → `..., incr(board.read_count)` 로 rewrite |
| `pt_has_aggregate()` 분기 | GROUP BY position 범위 체크 (`SORT_SPEC_RANGE_ERR`), host var 그룹 조건 금지, `WITH ROLLUP` ⊕ `GROUPBY_NUM()`, aggregate 인수 체크 (`COUNT(*)`/`ROW_NUMBER()`/... 외엔 arg_list 필수), `MEDIAN ... OVER(ORDER BY)` 금지, `PARTITION BY NULL` → `ORDER BY 3` 으로 rewrite |
| `pt_has_analytic()` 분기 | (별도 PPT 참조) |
| `pt_check_order_by()` | 불필요한 ORDER BY 제거 (`select count(*) from athlete order by code` → `… from athlete`), position 범위 |
| `CONNECT BY` 분기 | hierarchical query (TBU per analysis doc) |
| SHOW statement | metadata 로 select_list rewrite |
| Derived query | as_attr_list ambiguous reference (`MSGCAT_SEMANTIC_AMBIGUOUS_REF_TO`), hidden column → `ha_<number>` 임시 PT_NAME 추가 |
| `CAST` expression | COLLATE modifier 검사, type coercion 가능 여부 (`MSGCAT_SEMANTIC_CANT_COERCE_TO` vs `MSGCAT_SEMANTIC_COERCE_UNSUPPORTED`) |
| **LIMIT rewriting** | (아래 별도) |
| `pt_semantic_type()` | type checking + constant folding (별도 PDF) |

### LIMIT → Numbering Expression rewrite

| 패턴 | 변환 결과 |
|---|---|
| `LIMIT` with `ORDER BY` | `ORDER BY … FOR ORDERBY_NUM() BETWEEN 1 AND N` |
| `LIMIT` with `GROUP BY` | `… GROUP BY … HAVING GROUPBY_NUM() BETWEEN 1 AND N` |
| `LIMIT` with `DISTINCT` (no order/group) | `… for orderby_num() <= N` |
| `LIMIT` with plain `WHERE` | `… where (inst_num() <= N)` |
| Aggregate + LIMIT | derived table 로 wrap 후 `where inst_num() <= N` |

### Search condition cut-off / fold-as-false

WHERE/HAVING/ON 절을 CNF-list 로 보면:
- `A AND B AND true` → `true` 항 제거 (cut-off)
- `A AND B AND C` 중 하나라도 `false` → 전체를 false PT_VALUE 로 대체
- DNF-list 면: `D OR C OR true` → 전체를 true PT_VALUE 로 대체

## 5. Type Checking + Constant Folding — `pt_semantic_type()`

```c
PT_NODE *pt_semantic_type(...) {
  /* type checking */
  parser_walk_tree(tree, pt_eval_type_pre, …, pt_eval_type, …);
  /* constant folding */
  parser_walk_tree(tree, pt_fold_constants_pre, NULL, pt_fold_constants_post, …);
}
```

### `pt_eval_type_pre()` (top-down, 노드 진입)

| node_type | 동작 |
|---|---|
| `PT_SPEC` | derived_table 이고 query 이면서 outer join 존재 시 `has_outer_spec` flag (folding 방지) |
| `PT_SORT_SPEC` | sort spec expression 이 query 이면 `is_sort_spec` flag |
| query nodes | LIMIT → Numbering Expression (위 표 참조) |
| `PT_EXPR` | **Recursive Expression** 처리 (Left: `GREATEST/LEAST/COALESCE`, Right: `CASE/DECODE`) — 같은 operator 가 트리 왼/오른쪽으로 반복되는 노드 묶음을 하나의 expression 처럼 일괄 타입 결정 |

### `pt_eval_type()` (bottom-up, 노드 종료)

| node_type | 호출 함수 |
|---|---|
| `PT_EXPR` | `pt_eval_expr_type()` (아래 6 단계) |
| `PT_FUNCTION` | `pt_eval_function_type()` |
| `PT_CREATE_INDEX`/`PT_DELETE`/`PT_UPDATE` | search condition cut-off |
| `PT_MERGE`/`PT_SELECT` | search condition cut-off + 하위 (values/select list) 로부터 타입 지정 |

### `pt_eval_expr_type()` — 6 단계

1. **Rule-based**: PT_PLUS, PT_MINUS, PT_BETWEEN*, PT_LIKE*, PT_IS_IN*, PT_TO_CHAR, PT_FROM_TZ, PT_NEW_TIME — operator 별 implicit conversion table.
2. **Expression definition based**: `pt_apply_expressions_definition()` → `pt_get_expression_definition()` 에 미리 정의된 explicit conversion table 에서 best_match 찾고 `pt_coerce_expr_arguments()` 로 인수 변환.
3. **Collation compatible 확인**: 문자열 타입에 대해 `pt_check_expr_collation()`.
4. **Host variable late binding**: 인수가 host var (PT_TYPE_MAYBE) 면 `pt_wrap_expr_w_exp_dom_cast()` 로 expected_domain CAST 삽입. `pt_is_op_hv_late_bind()` 가 후보 op 판정.
5. **자명한 결과 타입**: PT_BETWEEN, PT_LIKE, PT_RAND/RANDOM/DRAND/DRANDOM, PT_EXTRACT, PT_YEAR/MONTH/DAY, PT_HOUR/MINUTE/SECOND, PT_MILLISECOND, PT_COALESCE, PT_FROM_TZ.
6. **타입 정해진 host var 인수에 expected domain 설정**.

### Function type evaluation

C 클래스 도입 (`func_type` namespace). `func_all_signatures` 와 매칭하여 인수/반환 타입 지정. 두 경로:

- `pt_eval_function_type_new()` — `func_all_signatures` 사용 (C++ 도입 이후)
- `pt_eval_function_type_old()` — legacy

### Constant Folding — `pt_fold_constants_post()`

`PT_EXPR` 와 `PT_FUNCTION` 가 모두 상수 인수만 가지면 평가 → 하나의 `PT_VALUE` 로 치환. `pt_fold_constants_pre()` 는 `benchmark()` 함수 호출이 있으면 CAS 에서 fold 되지 않고 executor 까지 가도록 PT_LIST_WALK 로 자식 진입 차단.

## Cross-references

- [[components/semantic-check]] — 기존 component 페이지
- [[qp-analysis-parser]] — 직전 단계
- [[qp-analysis-rewriter]] — 직후 단계 (`mq_translate`)
- 원문 보존: `.raw/qp-pdfs/semantic_check_*.pdf` (전체 텍스트 추출본 `.txt` 동봉)
