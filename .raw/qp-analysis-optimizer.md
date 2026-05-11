---
source: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-optimizer"
source_type: jira-wiki
title: "analysis-for-qp-optimizer"
slug: qp-optimizer
fetched_at: 2026-05-11
captured_via: playwright
language: ko
domain: cubrid-query-processing
attachments:
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/142_optimizer1.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/143_QO_ENV.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/144_optimizer_pre_info.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/155_optimizer_planner.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/156_optimizer_planner2.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/157_optimizer_planner3.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/158_optimizer_planner4.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/159_optimizer_planner5.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/160_optimizer_planner6.jpg"
---

# analysis-for-qp-optimizer

overview : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-

## OPTIMIZER

RDBMS의 query에는 어떤 조건의 데이터를 추출해야 되는지는 명확하나, 어떻게 scan하여 추출해야 되는지에 대한 정보는 없다.
OPTIMIZER에서는 어떻게 scan 할 것인가를 정하는 단계이며, 그 것은 **join의 순서, join의 방법, index의 선택** 에 대한 최적의 scan 방법을 결정하는 일이다.

## optimizer 수행 방식

optimizer는 xasl generator와 같이 수행되며, 아래와 같은 함수 호출을 통해 이루어진다.

```
parser_generate_xasl
parser_walk_tree (parser, node, parser_generate_xasl_pre, NULL, parser_generate_xasl_post, &xasl_Supp_info);

parser_generate_xasl_post
parser_generate_xasl_proc
pt_plan_query
qo_optimize_query <== optimization
pt_to_buildlist_proc <== xasl generation
```

`parser_walk_tree()` 함수를 사용하여 post order 방식으로 순회하며 `parser_generate_xasl_post()` 함수를 호출하여 진행된다.

OPTIMIZATION이 수행되는 곳은 `qo_optimize_query()` 이며, 이후 생성된 PLAN을 사용하여 `pt_to_buildlist_proc()` 에서 XASL을 생성한다.
`PT_NODE`의 type이 `PT_SELECT`일 때 수행되며, post order로 순회한다는 것은 마지막 node부터 top node로 순회하는 방식을 이야기한다.
이러한 방식 때문에 sub query의 경우 먼저 PLAN 및 XASL이 생성되고, 상위 XASL이 생성될때 하위 XASL을 링크하는 구조이다.

```sql
SELECT 1
   FROM TBL a, (SELECT COL1 FROM TBL2 ... ) b
....
```

위 쿼리를 예시로 설명하면 sub query인 `(SELECT COL1 FROM TBL2 ... )` 부분이 독립적으로 먼저 XASL 생성되며, 이후 main 쿼리에서 XASL 생성될 때 해당 in-line view의 XASL을 연결하는 방식으로 진행된다.

## 주요 용어

OPTIMIZATION을 진행하기 위해 QUERY GRAPH를 생성하는데 `QO_ENV` 객체에 담기게 된다. 해당 정보의 용어가 일반 DBMS의 용어와 다르며 아래와 같다.

- **SEGMENT** : 쿼리에 사용된 컬럼들
  - **final segment** : 최종 결과 값이 되는 컬럼 (select list)
- **NODE** : 쿼리에 사용된 table (spec, class)
- **TERM** : predicate (조인 조건, 조회 조건)
  - **edge** : table간 join 조건
- **PARTITION** : 서로 edge가 없는 독립된 node 그룹, optimization을 나눠서 진행하기 위해 구분한다.
- **EQCLASS** : edge가 되는 join 조건들의 모음. merge join을 하기 위해 sort plan을 생성하는데 사용된다.

## OPTIMIZER 진행단계

![optimizer overview](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/142_optimizer1.jpg)

위 그림에서 확인 할 수 있듯이 OPTIMIZER는 크게 두가지 단계로 나뉘어 실행된다.

1. query를 분석하여 node, term, segment 등의 정보를 생성하여 `QO_ENV` 객체에 저장한다.
2. 그 정보들을 바탕으로 가능한 PLAN을 서로 비교하여 최적화된 PLAN을 찾는다. 이 때 `QO_PLANNER` 객체를 사용하여 가능한 plan을 서로 비교하여 최적의 PLAN을 생성한다.

## 최적화 관련 정보 생성

![QO_ENV](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/143_QO_ENV.jpg)

위 그림은 parser tree에서 `QO_ENV` 객체가 어떻게 생성되는지 나타낸 것이다. `set optimization level 513` 으로 설정시 상단에 `QO_ENV` 정보를 확인할 수 있다.

### SEGMENT

query에서 사용된 column 정보이다. 위 그림에서는 `col1`과 `col2`가 선택되었다. 함수 기반 인덱스의 경우 column이 포함된 expression이 지정된다. (예: `seg1 : nvl(col1,1)`)
final segment는 마지막으로 조회되는 column이며, 위 그림에서는 select list에 있는 `col1`,`col2`의 순번이 bit set으로 표현된다.

### TERM

query에서 사용된 predicate 정보이다. 위 그림에서는 `col1=1` 과 `col2=1` 에 대해서 생성되었다. term은 term의 class에 따라서 명시된 predicate가 아닌 dummy term이 추가될 수 있다.
term의 class에 대해서는 아래 자세한 설명을 하겠다.

### NODE

query에서 사용된 table 정보이다. 위 그림에서는 `tbl a`가 생성되었다.
in-line view의 경우 sub query의 포함된 table 정보는 따로 optimization 하므로 생성되지 않고, in-line view를 하나의 list node로 인식하여 생성된다.

### PARTITION

node간의 join term이 없어 분리가 된 경우 각각 partition으로 지정되고, optimization을 따로 진행하며, 이후 두 partition을 카티션 곱으로 병합한다.

```sql
select 1
   from tbl1, tbl2, tbl3, tbl4
where tbl1.col1 = tbl2.col1
   and tbl3.col1 = tbl4.col1
```

위 쿼리의 경우 `tbl1`,`tbl2`가 partition1로 지정되고, `tbl3`,`tbl4`가 partition2로 지정된다. 각각에 대해서 optimization 진행후 생성된 plan을 카티션 곱으로 병합하는 하나의 plan으로 생성한다.

### EQCLASS

join term에 대한 각 노드별 segment 값을 갖는다.

```sql
select 1
  from tbl a, tbl b, tbl c
 where a.col1 = b.col1
    and b.col1 = c.col1
```

위와 같은 쿼리의 경우 eqclass는 `|col1[0]|col2[2]|col3[3]|` 값을 가진다. `[0]`의 의미는 첫번째 node 번호를 가르킨다.
eqclass를 통해서 node간 join term의 segment를 확인할 수 있으며, 이 정보는 sort merge join plan을 생성할 때 어떤 segment로 정렬을 해야되는 지 찾는데 사용된다.

### SUB QUERY

query에 포함된 sub query의 정보입니다. 위에서 설명한대로 sub query는 그 상위 query보다 먼저 독립적으로 optimization이 수행되며, select list에 있어 column 처럼 사용되어도 segment로 생성하지 않습니다.
in-line view의 경우는 하나의 node로 생성이 됩니다.
sub query 정보는 이후 XASL GENERATOR에서 XASL node간 연결을 진행할 때 사용됩니다.
sub query를 두 가지 종류로 구분하고 있으며, 그 이유는 scan 순서의 차이 때문입니다.

- correlated sub query : `select ( select 1 from tbl1 a where a.col1 = main.col1) from tbl2 main`
- uncorrelated sub query : `select ( select 1 from tbl1 a where a.col1 = 3) from tbl2 main`

correlated sub query의 경우 외부 참조되는 node의 scan이 먼저 진행되어야 scan이 가능하며, uncorrelated sub query는 가장 먼저 수행도 무방한 sub query 입니다.

## TERM의 CLASS INFO

TERM의 경우 OPTIMIZATION을 진행하는데 중요한 정보이기 때문에 가장 복잡한 정보이기도 합니다.
TERM의 성질을 나타내는 분류이다. 크게보면 JOIN에 관련된 TERM인지 조회조건(SARG)인지를 구분하며 저장되는 정보는 아래와 같다.

| TERM CLASS | VALUE | PATH | EDGE | FAKE | NUM |
|------------|------:|-----:|-----:|-----:|----:|
| QO_TC_PATH | 0x30 | 1 | 1 | 0 | 000 |
| QO_TC_JOIN | 0x11 | 0 | 1 | 0 | 001 |
| QO_TC_DEP_LINK | 0x1c | 0 | 1 | 1 | 100 |
| QO_TC_DEP_JOIN | 0x1d | 0 | 1 | 1 | 101 |
| QO_TC_DUMMY_JOIN | 0x1f | 0 | 1 | 1 | 111 |
| QO_TC_SARG | 0x02 | 0 | 0 | 0 | 010 |
| QO_TC_OTHER | 0x03 | 0 | 0 | 0 | 011 |
| QO_TC_DURING_JOIN | 0x04 | 0 | 0 | 0 | 100 |
| QO_TC_AFTER_JOIN | 0x05 | 0 | 0 | 0 | 101 |
| QO_TC_TOTALLY_AFTER_JOIN | 0x06 | 0 | 0 | 0 | 110 |

저장되는 정보를 확인해 보면 각 BIT 자리수에 따라 PATH, EDGE, FAKE TERM으로 구분한다. 3개 모두 JOIN과 관련있는 정보이다.

- **PATH** : indirect relations between data-elements.
  큐브리드의 객체지향을 구현하기 위해 컬럼에 row들을 담을 수 있는데 그 데이터를 조회할때 PATH TERM이 생성된다.
- **EDGE** : direct relations between data-elements
  RDBMS에서 join 조건에 해당하는 term이다. SQL 표준방식으로 ON 절 뒤에 올 수도 있고 WHERE절의 조회조건과 같이 작성될 수 있다.
  > 참고: optimization에서 SQL 표준으로 작성되었다고 하는 것은 사용자가 최초 작성한것을 의미하는 것이 아니고, query rewrite에서 변경되어 optimizer에 최종 전달된 사항을 말한다.
  > query rewrite에서는 inner join은 WHERE 절을 사용하는 묵시적인 방법으로 재작성하고, outer join은 SQL 표준 방식으로 재작성한다.
- **FAKE** : 실제 조인 predicate가 없는데 필요에 의해 생성해주는 join term이다.

TERM CLASS는 `qo_analyze_term()` 함수에서 평가되며 각각의 TERM CLASS의 설명은 아래와 같다.

### QO_TC_PATH

PATH join의 경우 ORDBMS의 기능으로 외부에서는 거의 사용되지 않으므로 여기서는 어떠한 것인지 예시를 통해 확인하고 이후 분석에서는 제외한다.
아래 예제쿼리는 카탈로그 테이블 `DB_USER`를 조회 할 때 PATH TERM이 생성되는 CASE이다.

```sql
set optimization level 513;
select /*+ recompile */ x.name from db_user as u, TABLE(u.direct_groups) as g(x) where g.x.name = 'PUBLIC';
```

```
Join graph edges:
term[0]: table(0) -> g node[1] (sel 1) (dep term) (inner-join) (loc 0)
*term[1]: g node[1] x -> db_user node[2] (sel 0.5) (path term) (mergeable) (inner-join) (loc 0)*
Join graph terms:
term[2]: g.x.[name]='PUBLIC' (sel 0.5) (sarg term) (not-join eligible) (indexable name[2]) (loc 0)
```

일반적인 RDBMS에는 없는 join 방법으로 메뉴얼에도 해당 내용은 없다. 위 예시는 `db_user` 테이블의 `direct_groups` 컬럼이 `set of db_user` type으로 설정되어 있어, 해당 컬럼에 db_user의 data row의 oid가 저장되는 구조이다.
아래와 같은 방법으로 조회를 할 것이다.

1. `db_user.direct_groups` 데이터 조회하여 `g` 노드 생성 ==> DEP_LINK_TERM
2. `g` 노드에 추출된 oid로 `db_user` 조회  ==> PATH TERM

추출된 oid를 row로 변환할 때 필요한 컬럼과 테이블의 관계를 표현한 term이라고 생각하면 되겠다.

### QO_TC_JOIN

predicate로 작성된 join term이다. predicate에 다른 node의 segment가 두개일때 해당 class로 선택된다.

### QO_TC_SARG

한개의 node에 대한 조회조건이다. 일반적인 조회 조건이라고 생각하면 된다.

```sql
select 1
  from tbl1 a, tbl2 b
 where a.col1 = b.col1  -- QO_TC_JOIN
    and a.col2 = 1       -- QO_TC_SARG
    and b.col2 = 1       -- QO_TC_SARG
```

### QO_TC_AFTER_JOIN

outer join 시 `where` 절에 작성되는 predicate로 join 이후 평가한다.

### QO_TC_DURING_JOIN

outer join 시 `on` 절에 작성되는 predicate로 join시 평가하며, 조건에 맞지 않아도 결과가 조회되는 특징이 있다.

(to_do : 설명하지 않은 term_class 추가 작성 필요)

## OPTIMIZER 사전 정보 생성

![optimizer pre info](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/144_optimizer_pre_info.jpg)

위 그림은 optimizer 사전정보 생성시 주요 로직을 순차적으로 나타낸 것이다. 회색 박스안의 함수는 해당 로직의 시작 지점이므로 참고하기 바란다.
명시적으로 함수가 표기되지 않은 것은 `qo_optimize_helper()`에서 직접 수행된다.

### ENV 객체 생성 및 초기화

```
qo_optimize_query()
qo_env_init()
```

ENV 객체를 생성할때 node, term등의 개수를 parser tree를 통해 예측하여 메모리 할당을 미리한다.
아래와 같은 동작을 진행한다.

1. node, term, segment 개수를 예측함 (`qo_validate`)
2. node개수가 **64**를 넘으면 에러 발생
3. 예측된 개수를 활용하여 ENV객체의 node, terms, segs, eqclass, partition 항목 메모리 할당함.
4. `env->Nnodes, Nsegs, Nterms, Neqclasses` : 초기화시 미리 할당한 구조체 배열의 최대 자리수를 의미함.
5. `env->nnodes, nsegs, nterms, neqclasses` : node, term등을 생성하면서 실제 할당된 구조체 개수를 의미함.

### ENV.node 추가 및 segment 추가

```
build_query_graph()
build_graph_for_entity ()
```

1. `PT_NODE`의 `info.spec` 정보를 참조하여 `ENV.node`를 추가하며 통계정보를 참조하여 전체 row수와 page수를 저장한다. (`qo_add_node`)
   - `QO_NODE_NCARD` = table의 전체 row수
   - `QO_NODE_TCARD` = table의 전체 page수
   - derived table의 경우 `xasl->cardinality`를 활용하여 CARD를 저장한다. (`xasl->card`는 이전 optimization에서 계산된 값으로 보이며 이후 분석 필요)
2. `PT_NODE`의 `info.spec.referenced_attrs` 정보를 참조하여 segment 정보를 저장한다. (`referenced_attrs` 분석필요)
3. PATH 관련 정보가 있다면 해당 PATH의 segment와 term을 저장한다. RDBMS의 join 조건으로는 PATH는 생성되지 않고, 일반적인 사용법은 아니다.
   - PATH 관련 정보 : `entity->info.spec.path_entities`, `entity->info.spec.path_conjuncts`. `from tbl a, TABLE(a.col1)` 와 같은 사용에서 생성되나 어떻게 입력이 되는지는 분석 필요.

### DEPENDENT TABLE에 대한 DEPENDENT TERM 추가

DEPENDENT TABLE은 아래와 같은 경우이며 일반적인 RDBMS 사용에서는 correlated sub query가 해당합니다.

- `PT_IS_SET_EXPR` : table로 set type이 사용된 경우 (예: `from tbl a, TABLE(a.col1 or {1,2,3}) b(x)`)
- `PT_IS_CSELECT` : 사용안하는 type (we removed MERGE/CSELECT from grammar)
- `PT_DERIVED_JSON_TABLE` : json
- `PT_IS_SUBQUERY && info.query.correlation_level == 1` : correlated sub query

correlated sub query의 경우 sub query의 결과를 얻기위해 연관있는 node의 데이터가 필요하며, 이는 scan의 순서가 이미 결정되어 있음을 의미합니다.
현재 level의 쿼리에서 하위 쿼리와 조인 조건이 있음을 지정하기 위해서 subquery node의 `QO_NODE.dep_set`에 연관있는 NODE를 지정하고 FAKE TERM(`QO_TC_DEP_LINK`)을 생성합니다.

`qo_optimize_helper()`에서 호출되는 관련된 주요 함수:

- `qo_expr_segs()` : 입력 받는 `PT_NODE`를 walk하여 현재 진행하는 쿼리 LEVEL의 NODE와 일치하는 SEGMENT의 BITSET을 저장한다.
- `qo_seg_nodes()` : segment의 bitset 정보에 해당하는 node의 bitset 정보를 저장함.
- `qo_add_dep_term()` : `derived_node`의 `dep_set`에 `depend_nodes`를 설정하고 term을 생성하여 `QO_TERM_CLASS: QO_TC_DEP_LINK`, `QO_TERM_JOIN_TYPE: JOIN_INNER` 저장한다. 결정되는 join 순서에 맞게 head, tail node를 설정한다.

### ON COND TERM 추가

sql 표준으로 작성된 join문의 `ON` 절의 predicate를 term에 추가한다. `qo_analyze_term()`에서 term의 성질을 저장한다.

### PREDICATE TERM 추가

`WHERE`절의 PREDICATE TERM을 생성한다. 아래는 term 추가시 관련있는 주요 함수이다.

- `qo_add_term()` : TERM 정보를 추가한다.
- `qo_analyze_term()` : term의 class, join type, 인덱스 가능여부 등 대부분의 term의 정보를 판별하여 저장합니다.
  - term class : sarg term(조회조건), join term(조인조건)
  - selectivity : 선택도
  - indexable : 인덱스 스캔 가능 여부
  - 여기서 indexable은 인덱스와의 비교를 통해 가능여부를 확인 하는 것이 아니고, operation, LHS, RHS type으로 index scan이 가능한지 여부를 확인한다. (예: `OP == EQ` then indexable, `OP == NOT_EQ` then not indexable)
  - 실제 인덱스에 대한 비교는 `qo_discovery_indexes()` 및 `qo_generate_join_index_scan()`에서 진행된다.

### DUMMY JOIN TERM 추가

sql 표준 양식으로 작성된 테이블간의 관계에서 연속된 node에 join term이 없을 경우 dummy term을 생성한다.
이후 join 순서를 결정하는 permutation에서 join term이 없는 경우 더 이상 진행하지 않기 때문에 dummy term을 추가함으로써 계속 진행하려는 의도로 생각된다. (추가 의도에 대해서는 더 확인이 필요함)

```sql
from tbl1 a left outer join tbl2 b on a.col1 = b.col1
                  left outer join tbl3 c on a.col2 = c.col2
```

`a=>b`, `a=>c` 관계가 있으며 `b=>c`의 관계가 없다. 해당 관계를 dummy term으로 추가한다.

### OUTER JOIN에 대한 TERM CLASS 저장

```
qo_classify_outerjoin_terms()
```

`qo_analyze_term()`에서 이미 term의 상세 정보가 저장되었으며, 여기서는 outer join에 대해서 추가로 확인을 진행한다.

OUTER JOIN 관련 정보는 TERM CLASS, OUTER DEPENDENCY 정보이다.

**TERM CLASS**
- `QO_TC_DURING_JOIN` : OUTER JOIN 수행시 같이 수행되는 SARG 종류의 PREDICATE
- `QO_TC_AFTER_JOIN` : OUTER_JOIN 수행 이후 filter해야 되는 SARG 종류의 PREDICATE
- SARG : search argument의 약자로 일반적으로는 predicate를 의미하나, cubrid에서는 join term이 아닌 조회 term을 말한다. (예: `a.col1 = b.col1` : join term, `a.col1 = 3` : SARG term)

on절에 사용되는 predicate는 during_join에 해당하며, outer join 특성에 따라 평가 값이 참이 아니더라도 조회된다.
where 절에 사용된 predicate는 after_join에 해당하며, join 이후 조회해야 하는 항목이다.

```sql
from tbl1 a left outer join tbl2 b on  a.col1 = b.col1 and b.col2 = 2   -- QO_TC_DURING_JOIN
where nvl(b.col3,1) = 4                                                  -- QO_TC_AFTER_JOIN
```

위 예시는 where 절의 predicate가 `nvl()` 함수를 사용하여 null이 허용되는 경우이며, 해당경우는 join이후 평가된다.
만약 null이 허용되지 않는 경우라면 rewrite에서 outer join을 inner join으로 변경한다. (`qo_rewrite_outerjoin()`)

**OUTER DEPENDENCY**
`node.outer_dep_set`을 설정한다. 해당 정보는 permutation을 진행하면서 조인순서의 제약을 주게 된다.

```sql
select 1 from a left outer join b on a.col = b.col
```

위와 같은 쿼리에서는 `b` node의 `outer_dep_set`에 `a` node를 설정하며, join 순서 결정시 `b` node는 `a` node보다 먼저 선택될 수 없다.

### final segment 체크

`qo_add_final_segment()` : final segment 추가한다.
final segment는 최종 조회되는 값이다. select_list, group by, having와 connect by 관련 segment들이 포함된다. group by, having 등은 executor에서 scan 이후 후처리로 진행되기 때문에 final segment에 포함된다.

### predicate 정렬 및 edges 체크

```
qo_discover_edges()
```

edge term을 sarg term 앞으로 정렬하고, edge term과 sarg term을 selectivity 내림 차순으로 정렬한다.
`NODE.sargs` bitset 정보에 term index를 추가하고 node의 selectivity를 계산하여 입력하는데 공식은 다음과 같다. `(node's card * sarg term들의 selectivity)`
불가한 edge term이 있는지 체크를 한다 (추가분석 필요).

### EQ_CLASS 설정

```
qo_assign_eq_classes()
```

테이블의 JOIN TERM을 확인하여 동일한 값을 갖는 컬럼을 지정한다. (예: `t1 a, t2 b where a.col1 = b.col2` ==> `eq_class( a.col1, b.col2)`)
해당 정보는 SORT MERGE JOIN에서 정렬한 컬럼을 찾는데 사용된다.

### index 컬럼 매칭 정보 저장

```
qo_discover_indexes()
```

node가 가리키는 table에 생성되어 있는 index 정보와 term의 segment를 비교하여 인덱스 매칭이 되는 term index를 SEGMENT의 `index_terms` bit set에 저장한다.
대상이 되는 term class는 `QO_TC_SARG`로 join 조건은 포함되지 않는다. JOIN TERM의 경우는 index join 방법 PLAN 생성시 매칭 정보를 확인한다.

`index_entry` 구조체에 term과 index 컬럼의 매칭정보를 저장하는데 아래와 같다.

- `index_entry->seg_equal_terms[idx]` : term과 index column이 매칭되고 operation이 `=` 인 경우 (RANGE LIST도 예외적으로 1회 입력 가능)
- `index_entry->seg_other_terms[idx]` : term과 index column이 매칭되고 operation이 `=` 아닌 경우
- 'RANGE LIST'는 'IN' operation을 말함.

예시:

```sql
index(col1,col2)
where col1 = 3            -- index_entry->seg_equal_terms[1]에 해당 term bit 입력
    and col2 > 3          -- index_entry->seg_other_terms[2]에 해당 term bit 입력
    and col3 range (1 = or 2 =)
```

### PARTITION 정보 생성

```
qo_discover_partitions()
```

node간에 edge가 없는 그룹을 partition으로 구분한다.
plan 생성시 partition이 다른경우 optimization 각각 수행하여 각각의 PLAN이 생성되며, 이후 생성된 PLAN을 cartesian product로 병합한 하나 PLAN이 생성된다.

## OPTIMIZATION 수행

![optimizer planner](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/155_optimizer_planner.jpg)

OPTIMIZATION 두번째 단계는 `QO_ENV`를 포함하는 최적화 관련 정보를 활용하여 PLAN을 생성하는 로직이다.
위 그림은 주요 로직에 대한 순차적인 흐름을 나타낸 것으로 네모 박스의 두번째 줄에 작성된 함수들은 해당 로직의 시작 지점이므로 참고하기 바란다.

아래와 같은 단계로 수행된다.

1. **planner 객체를 생성한다.** planner 객체에는 candidate plan들을 저장하고, 각각을 비교하기 위해 사용된다.
2. **node별 best plan을 생성한다.** join term을 제외하고 각각의 node의 sarg term을 사용하여 full scan 혹은 index scan중 best plan을 찾는다.
   - cubrid는 index scan이 가능하며 무조건 index scan을 best plan으로 선택한다. 그러므로 index가 있는 node의 경우 어떤 index를 사용하여 scan할 지 결정한다.
3. **join순서를 선정하기 위한 permutation을 진행하기 전에 시작 node들을 선정한다.** permutation의 경우 수가 `node!` 이므로 경우의 수를 줄이기 위한 방안으로 생각된다.
4. **permutation을 진행한다.** 가능한 join의 순서로 plan을 생성하고 각각을 비교하여 최적의 plan을 찾는다. join method의 선택에 따른 plan들도 같이 비교된다.

위 두 단계는 각 partition마다 따로 수행되며, 각각 생성된 plan을 병합하여 하나의 plan으로 만든다.

### planner 객체 생성

![optimizer planner2](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/156_optimizer_planner2.jpg)

위 그림은 planner 구조체가 어떻게 생성되는지 나타낸 것이다.
planner 구조체는 크게 node info와 join info로 구성된다.

**node info**
각각의 node의 plan이 저장된다. 위 그림에서는 table a, b 그리고 c가 저장되며 sarg term을 사용한 개별 node의 best plan이 저장된다.
`best_no_order`에 정렬이 되지 않은 plan이 저장되며, `planvec`에는 sort merge join시 사용될 조인조건에 대해 정렬된 best plan이 저장된다.

**join info**
node의 개수에 따라 `2^node` 만큼 저장공간이 생성된다. 각각의 bit가 node간의 조합을 뜻한다. 위 그림에서는 table a, b, c의 조합이 어떻게 담길지 나타내고 있다.
가장 마지막 bit에 해당하는 공간은 모든 node의 조합에 대한 plan이 생성되므로 해당 공간이 `best_info`로 지정된다.

위와 같은 자료구조의 배열의 크기를 설정하기 위해 `planner->N, E, T, S, EQ, Q, P`의 값을 결정하는데 각각 `QO_ENV` 구조체의 `nnode, edge, nterm, nsegment, neqclass, subquery, partition`의 개수를 사용한다.
`planner->M`은 join info의 크기를 설정하기 위한 것으로 `2^node`만큼 설정된다. node 수가 증가함에 따라 메모리의 사용도 급격하게 증가하게 되는 구조이다.

### node별 plan 생성

![optimizer planner3](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/157_optimizer_planner3.jpg)

위 그림에서 왼쪽 query에 빨간색 박스로 표시된 term이 sarg term에 해당한다. 해당 쿼리는 3개의 node가 있으므로 3개의 각 node의 best plan이 생성된다.
왼쪽 아래는 각 node에 대해서 best plan을 생성하는 과정을 간략하게 표현한 것이다.
각각의 node에 해당하는 sarg term을 사용하여 index가 없는 경우 full scan을 선택하며 index가 있는 경우 각각의 index에 대한 plan을 생성하여 비교하고 가장 최적의 정보를 `node_info`의 `best_no_order`에 저장한다.

### node별 ordered plan 생성

![optimizer planner4](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/158_optimizer_planner4.jpg)

ordered plan을 생성하는 이유는 sort merge join의 plan을 생성할 때 해당 plan을 가져다 사용하기 위함이다.
정렬이 추가된 점과 eqclass의 개수에 따라 `planvec`에 저장되는 점을 제외하고는 위와 같은 방식으로 동작한다.

### first node 선별

![optimizer planner5](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/159_optimizer_planner5.jpg)

first node를 선별하는 것은 permutation의 경우의 수를 줄이기 위함이다.
위 그림에서 `Tcard`와 `Ncard`가 나오는데 그 의미는 아래와 같다.

- `Tcard` : scan시 예측되는 page수
- `Ncard` : scan시 예측되는 row수

`dep_set`, `outer_dep_set` node는 선행하는 node가 반드시 있으므로 제외하고 derived table은 index scan을 할 수 없으므로 추가된다.
나머지 node에 대해서 위 그림의 오른쪽 순서도와 같이 선별이 된다. 여기서 cost는 scan시 발생될 비용을 말하며, card는 cardinality의 줄임말로 예상되는 row수를 이야기한다.
위 그림의 오른쪽 예시는 비효율적으로 시작 node가 선정될 수 있는 예시이며, 대입하여 계산시 a node만 first node로 선정되어 `a->b` 순서만 가능하다.
이때 b node에는 인덱스가 없으므로 `b->a` 순서보다 좋지 않은 실행 결과를 가져올 수 있다.
이는 index scan 가능 유무에 대한 확인을 오른쪽 로직에서만 확인하기 때문에 발생한다.
또한 first node의 선정시 cost의 비교로직이 적절하지 않다고 생각되며 card(예측되는 row수)를 기준으로 삼아야 할 것이다.
이러한 이유로 이후 로직의 수정이 필요한 부분이라고 생각된다.

### permutation

![optimizer planner6](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/160_optimizer_planner6.jpg)

join의 순서와 join method에 대한 최적의 plan을 찾기 위해서 각 node의 순열 경우의 수만큼 plan을 생성하며 비교한다.
두 node의 join plan을 생성하고, 그 이후 나머지 node와 join하는 plan을 생성한다. 예를 들어 `A->B->C`의 순서라면 `A->B`를 먼저 생성하고 `(A,B)->C`의 plan을 생성한다.
위 그림의 query의 node수는 총 3개이며 join term이 존재하는 조합은 `(A,B)`와 `(B,C)`이다. `(A,C)`의 join term이 없는 것을 확인 할 수 있다.

오른쪽 아래가 permutation이 진행되는 예시이며, 아래와 같이 동작된다. first node는 a, b, c 모두 선정되었다고 가정한다.

1. `A->B`에 대한 plan을 생성하고 해당 plan을 `join_info`의 `011` bit에 해당하는 공간에 저장된다. cost는 300이 나왔다고 가정한다.
2. `(A,B)->C`에 대한 plan을 생성한다. bit가 `111`인 `join_info`의 마지막 공간에 저장된다. cost는 500이라 가정한다.
3. `A->C`는 edge(join term)이 없다. 더이상 진행하지 않는다.
4. `B->A`에 대한 plan을 생성하였고 cost가 400이라 가정한다. `join_info`의 `011` 공간에는 이미 `A->B`의 plan이 들어가 있으므로 두 plan을 비교하여 cost가 높은 `B->A` plan은 버려진다. 이후 작업인 `B->A->C` 역시 진행되지 않는다.
5. `B->C`는 cost가 600 나왔고, `join_info`의 `111`에 저장된 `A->B->C` plan보다 cost가 높으므로 생성된 plan은 버려지고, 이후 관련 순열은 진행되지 않는다.
6. `C->A`는 edge(join term)이 없다. 더이상 진행하지 않는다.
7. `C->B`는 cost가 100이다. `join_info`에 해당 plan을 저장하고 계속 진행한다.
8. `(C,B)->A` cost가 300이 나왔고, `111`에 해당하는 `A->B->C` plan보다 낮은 cost이므로 해당 공간은 이 plan으로 대체된다.

permutation이 완료되면, `best_info` 값이 best plan을 가지고 있게 된다. 만약 best info의 값이 없으면, optimization은 실패하게 되고 기본 plan으로 수행된다.

to_do : join method, cost 산정, plan compare, plan 객체
