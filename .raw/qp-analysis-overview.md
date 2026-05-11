---
source: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-"
source_type: jira-wiki
title: "analysis for Query processing"
slug: query-processing-overview
fetched_at: 2026-05-11
captured_via: playwright (mcp__playwright__playwright_navigate + evaluate)
language: ko
domain: cubrid-query-processing
labels: [cubrid]
owner: "박세훈"
last_editor: "채광수"
last_edited: "2026-03-03"
sub_pages:
  - parser: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-parser"
  - semantic_check: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-resolve-names"
  - rewriter: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-rewriter"
  - optimizer: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-optimizer"
  - xasl_generator: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-xasl-generator"
  - executor: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-executor"
  - tempfile: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-tempfile"
attachments:
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/131_query%20process.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/141_parser.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/136_pre%20fetch.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/140_rewriter.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/135_optimizer.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/137_xasl%20generator.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/138_executor.jpg"
---

# analysis for Query processing — overview

> Hub page. Sub-pages (originals on internal JIRA wiki):
> - parser : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-parser
> - semantic check : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-resolve-names
> - rewriter : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-rewriter
> - optimizer : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-optimizer
> - xasl generator : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-xasl-generator
> - executor : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-executor
> - tempfile : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-tempfile

CUBRID는 open source DBMS입니다. CUBRID에 더 많은 contributor가 생겼으면 하는 마음에서 Query Process의 개괄적인 설명을 담습니다.

## Overview — Query Process란?

Query Process(QP)는 DBMS의 입력값인 SQL을 낮은 수준의 명령으로 변환하고 그것을 실행하는 전체 작업을 말합니다. 여기서는 SQL이 낮은 수준의 명령으로 어떻게 변환 되는지에 대한 일련의 작업을 개괄적으로 설명하겠습니다.

SQL에서 가정 먼저 진행 되어야 하는 것은 TEXT로 작성된 SQL을 parse tree라는 구조로 만드는 것입니다. 이 작업은 PARSER에서 진행되는데, CUBRID는 `PT_NODE` 구조체를 반복적으로 사용하여 SQL을 parse tree로 변환합니다. 이 단계에서 자연스럽게 syntax check가 진행되고, 이것은 문법적인 체크로 오타나 잘못된 예약어등을 체크합니다. 그리고 SEMANTIC CHECK를 진행하는데, 이 것은 변환된 parse tree에서 지정된 NAME이 DBMS에 등록된 것인지 체크합니다. 예를 들면 작성된 테이블명이나 컬럼명등이 존재하는 것인지 체크합니다. 그리고 TYPE CHECK도 진행되는데 이는 SQL 안에서 TYPE 매칭이 호환 가능하지 체크하고 CAST하는 역할을 합니다. SYNTAX CHECK와 SEMANTIC CHECK가 완료되면 입력된 SQL TEXT가 실행가능한 parse tree임이 확인 된 것입니다. (물론 실행중에 에러가 발생되는 케이스도 상당히 많습니다.)

다음으로 OPTIMIZER가 parser tree를 최적화하고 PLAN을 생성합니다. parse tree를 최적화하는 것을 QUERY REWRITE 혹은 TRANSFORMATION이라고 합니다. 좋은 성능을 위해 SQL을 다시 작성한다고 생각하면 됩니다. 동일한 데이터를 조회하는 SQL은 다양한 형태로 작성될 수 있습니다. 그렇기 때문에 가장 효과적인 방안으로 변환을 하는 것입니다. 여러 재작성 방법이 있는데 조회조건을 sub-query 안으로 넣어주는 predicate push등이 이에 해당합니다. 그 다음은 PLAN을 생성하는 것입니다. PLAN에는 어떻게(HOW) DATA를 조회할 것인가에 대한 정보가 있습니다. SQL은 어떤(WHAT) 데이터를 조회할 것인지에 대해서만 명시되어 있기때문에 실제 데이터를 조회 하기 위해서 이 단계가 필요합니다(이 단계때문에 개발자가 어떻게 DATA를 조회할지에 대한 정보를 알지 못해도 됩니다). 여기서 어떻게 조회하는가에 대한 정보는 JOIN 테이블의 SCAN 순서, JOIN METHOD, SCAN METHOD등의 정보를 말합니다. 해당 정보가 PLAN 저장 됩니다.

XASL generator에서 PLAN과 parse tree의 정보를 종합하여 실행계획 정보를 생성는데 이것은 EXECUTOR에서 바로 실행이 가능한 형태입니다. CUBRID는 이것을 **eXtentional Access S Languege (XASL)** 형태로 저장합니다. XASL에는 어떻게, 어떤 데이터를 조회할지 명시되어 있습니다.

마지막으로 EXECUTOR에서 XASL을 해석하여 `heap_next()`, `index_scan_next()` 등의 operation을 실행합니다. 이것들은 데이터를 조회하거나 저장하는 낮은 레벨의 명령입니다. 이로써 Query Process가 완료되었고 DBMS 사용자는 원하는 데이터를 얻게 되었습니다.

![QP overview](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/131_query%20process.jpg)

CUBRID의 query Process를 간략하게 나타낸 것이며, 각 회색 박스안에 두번째 줄에 적힌 함수가 각 단계의 시작지점이므로 참고하기 바랍니다.

QP(Query Process)의 작업을 요약하면 아래와 같습니다.

1. **PARSER** 단계를 통해 parse tree 구조체(`PT_NODE`)가 생성
2. **SEMANTIC CHECK**를 통해 정보가 추가된 parse tree가 생성
3. **REWRITER** 단계를 통해 최적화된 parse tree로 변형
4. **OPTIMIZER**에서 어떻게 조회 할지에 대한 PLAN을 생성
5. **XASL GENERATOR**에서 실행가능한 XASL를 생성
6. **EXECUTOR**에서 XASL을 실행하여 원하는 데이터 조회

## PARSER

parser는 text인 SQL를 parsing 하여 parse tree를 생성합니다. 오픈소스 **BISON**을 사용하며, `PT_NODE` 구조체를 반복 사용하여 tree를 구성합니다.

아래는 간단한 쿼리에 대해 parser가 parse tree를 `PT_NODE` 구조체로 나타낸 예시입니다.

![parser PT_NODE](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/141_parser.jpg)

위 그림에서 하나의 BOX는 `PT_NODE` 구조체를 나타냅니다. 파란색 부분은 `PT_NODE.node_type` 이고, 하늘색 부분은 `PT_NODE.info` 공용체를 나타낸 것입니다. `PT_NODE` 구조체는 node_type에 따라 공용체인 info의 선택되는 변수가 달라집니다. 예를 들면 `PT_SELECT`는 `info.query`를 사용하고, `PT_EXPR`은 `info.expr`을 사용합니다. 다양한 parse tree의 구조를 확인하기 위해서는 `parser_main()` 함수 실행 이후 결과에 대해서 `PT_NODE` 구조체를 확인해 봐야하는데, text 기반인 gdb보다는 DDD(display data debuger)를 사용하는 것을 추천합니다.

parsing은 BISON통해 진행됩니다. BISON의 동작원리에 대한 이해는 아래 링크를 통해 확인하기 바랍니다.
https://www.gnu.org/software/bison/manual/html_node/index.html

새로운 sql 구문의 추가나 변경시에 parser의 수정이 있어야 하는데 `csql_lexer.l`, `csql_gramar.y` 파일의 수정이 필요합니다. `csql_grammar.y`에서 grammar의 action 부분에는 `PT_NODE`를 생성하는 로직이 포함되어 있으니 관련 있는 키워드를 통해 로직을 확인하는 것이 중요합니다.

parse tree를 생성하는 과정에 자연스럽게 syntax check가 진행되며, 간단한 rewrite도 진행됩니다. 위 예시는 조회조건의 변경이 있었습니다.

생성된 parse tree를 체크하기 위해 tree를 탐색해야 하는데 `parser_walk_tree()` 함수가 이를 수행합니다. pre-order와 post-order 함수를 인자에 추가 할 수 있어 tree를 탐색하며 적절한 동작을 할 수 있습니다. 또한 node_type에 따라 어느 info의 공용체가 사용되는가에 대해서는 `parser_walk_tree()`에서 적용되는 apply 함수를 확인하면 되는데, `pt_init_apply_f()` 함수에서 어떠한 apply 함수가 적용되는지 확인할 수 있습니다. 예를 들면 `PT_SELECT`의 경우 `pt_init_apply_f()` 함수에서 확인하면 `pt_apply_select()`를 사용하는 것을 알수 있는데 해당 함수에서 info의 어느 변수를 사용하는지 확인이 가능합니다.

> parser : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-parser

## PRE FETCH

타 DBMS의 차이를 보이는 것은 'PRE FETCH' 단계가 있는 것입니다. 이 단계를 SEMANTIC CHECK의 한 부분으로 생각해도 되겠습니다.

CUBRID는 CAS와 CUBRID server가 분리되어 있는 3-tier 구조이기 때문에 TABLE의 스키마정보등 meta-data에 대한 client(CAS)와 server(CUB_SERVER)의 동기화가 필요합니다. SQL에 포함된 테이블 정보에 대해서 CAS가 cub_server에 요청하여 데이터를 받고 CAS의 work space공간에서 해당 데이터를 cache하여 사용합니다.

![pre fetch](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/136_pre%20fetch.jpg)

위 그림은 cub_server와 CAS간에 meta-data를 주고 받는 흐름을 나타낸 것입니다.

- QUERY가 입력되면 `pre_fetch`에서 CAS의 work space에 해당 table이 cache되어 있는지 확인
  - cache되어 있음 : CHN(cache coherency number) 번호를 server에 전송하여 최신 object임을 확인. 아닐경우 server로부터 데이터 가져옴
  - cache되어 있지 않음 : server로부터 object 가져옴
- CAS의 work space에 `Classname_cache`라는 전역변수에 해당 object를 cache함
- `Ws_commit_mops`에는 해당 transcation에 관련 있는 object를 linked list로 관리함.
  - commit 시 : `Ws_commit_mops`의 object 삭제
  - rollback 시 : `Classname_cache`에서 해당 transaction의 관련 있는 object를 decache함.

rollback시 cache된 object를 삭제하는 것은 DDL을 통해 CAS의 저장된 object만 변경이 되어 있을 수 있기 때문이다. DDL의 경우 (더 분석이 필요하지만) cub_server로 DDL 구문을 전달하는 것이 아니고, CAS에서 해당 object를 변경하고 그것을 flush하여 overwrite하기 때문에 commit 전에는 cub_server와 CAS의 object가 다를 수 있다. 하지만 DDL이 없는 경우에도 CAS의 object를 decache하는 점은 비효율적인 로직이라 생각된다.

PRE FETCH는 단순히 테이블 정보와 같은 meta-data를 가져오는 단계입니다. CUBRID가 가지는 3-tier 구조의 장점으로 client(CAS) side 수행을 통한 부하 분산을 생각할 수 있는데, 그것을 위해 해당 코드에 대한 분석이 필요하다는 생각입니다.

## SEMANTIC CHECK

SEMANTIC CHECK 단계에서는 크게 3가지 작업을 진행합니다.

1. **이름 해석 (name resolution).** 테이블명/컬럼명 등의 정보를 체크하고 관련 정보를 추가합니다. 이렇게 schema 정보에서 얻어와야 하는 정보는 parse tree 생성시 `PT_NODE`의 node_type이 `PT_NAME`으로 생성됩니다. 즉 node_type이 `PT_NAME`인 node에 대해서 체크하고 정보를 추가하는 작업이 필요합니다. `pt_resolve_names()` 함수에 관련 작업이 진행됩니다.
2. **type 체크.** 스키마의 등록된 type과 이에 대칭되는 상수 혹은 다른 컬럼의 type이 호환가능한지 확인하고 필요에 따라 `CAST` 함수를 추가해줍니다. type check에는 문자열의 charset과 collation도 포함됩니다. `pt_semantic_type()` 함수에 관련 작업이 진행됩니다.
3. **constant fold.** 상수만으로 구성된 함수나 연산등을 EXECUTOR 단계에 가기 전에 미리 처리합니다. 예를 들면 `select 1+2 from dual`에서 `1+2`의 연산을 미리 계산하여 `3`으로 변환합니다. CUBRID에서는 CAS에서 따로 수행되기 때문에 DB SERVER의 부하를 줄여주는 역할을 합니다. `pt_fold_constants_post()` 함수에 관련된 주요 소스코드가 있습니다.

그 외에도 많은 체크나 변환이 포함되어 있습니다만 주요 내용을 요약하면 SEMANTIC CHECK는 스키마와 같은 meta data를 parse tree에 추가하고 그 과정에서 문제시 semantic error를 발생하는 단계입니다.

> semantic check : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-resolve-names

## REWRITER (QUERY TRANSFORMATION)

REWRITER 단계에서는 SQL을 재작성합니다. parse tree에서 특정 패턴을 인식하고 그 구조를 변경하는 작업입니다. 주로 SQL의 성능을 향상시키기 위한 작업입니만 이후 단계의 작업을 위해 변환하는 작업도 있습니다. 많은 변환 패턴이 있지만 몇가지만 소개하겠습니다.

- predicate의 형식을 CNF(Conjunctive Normal Form)로 변환
- view를 view spec으로 derived table 변환
- predicate의 상수 변수를 host 변수로 치환
- uncorrelated subquery를 join으로 변환
- sub-query 밖의 관련 조회조건을 안으로 이동
- 그 외 다수의 변환이 있음

여기서 CNF변환과 view의 변환은 이후 작업 진행을 위해 진행되는 재작성입니다. 조회 조건을 CNF로 변환하는 것은 인덱스 스캔의 대상을 효과적으로 찾기 위함입니다. CNF는 논리곱 표준형으로 'AND'로 묶인 조회조건입니다. 아래 예시를 보면 CNF와 DNF의 차이를 확인할 수 있습니다.

```
(col1=1 or col2=1) AND (col3=1 or co4=1)       ==> CNF
(col1=1 and col2=1) OR (col3=1 and col4=1)     ==> DNF
```

CUBRID는 모든 조회 조건을 CNF로 변환하려는 시도를 하며, CNF로 변환되거나 작성된 조회조건만 인덱스 스캔의 대상이 됩니다. 이론상 모든 DNF는 CNF 변환이 가능하지만 조회조건의 개수가 늘어 날 수 있는데 CUBRID는 한개의 조회조건의 변환된 결과가 **100개를 넘게되면 해당 조회조건을 변환하지 않습니다**.

> DNF로 작성된 predicate를 CNF로 변환시 상황에 따라 predicate의 수가 상당히 늘어나게 되는데 그 수가 100개를 넘어가면 변환을 진행하지 않는다. 큐브리드는 CNF로 구성된 predicate에서 optimization이 가능한데, 이는 CNF로 변환이 되지 않은 predicate는 인덱스 스캔이 불가함을 의미한다.

![rewriter](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/140_rewriter.jpg)

위 그림에서 빨간색 박스부분을 보면 predicate가 CNF 변환이 적용된 것을 볼 수 있습니다. 위 그림의 predicate는 `col1=1 and col2=1` 으로 그 자체로 CNF이므로 predicate의 자체는 변경되지 않았지만, `AND`,`OR`와 같은 logical operation의 변경이 있었습니다. parser에서는 logical operation을 하나의 `PT_NODE`로 표현했지만, 변환된 이후에는 `next` (AND)와 `or_next` (OR) 포인터를 사용하는 구조로 변경되었습니다. 이러한 구조로 변경 되었기 때문에 이후 predicate 검색은 아래와 같이 수행됩니다. `PT_AND`와 `PT_OR` operation을 가지는 predicate는 cnf 변환이 안된 것입니다.

```
while PT_NODE.next
   predicate 처리
   while PT_NODE.or_next
     predicate 처리
```

이 이외에도 다양한 재작성이 진행됩니다. `qo_optimize_queries()` 함수에 대부분의 재작성 소스코드가 작성되어있습니다. 해당 함수의 진행 이전과 이후의 tree를 비교해여 어떠한 재작성이 진행되었는지 확인 할 수 있습니다. 또한 plan 정보에서 재작성된 SQL을 text 형태로 확인 할 수 있습니다.

rewriter에 대한 더 자세한 내용은 아래 링크를 확인하기 바란다.
http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-rewriter

## OPTIMIZER

OPTIMIZER는 어떻게 DATA를 검색할 것인가에 대한 정보를 찾고 비교하여 최적의 PLAN에 저장합니다. 그 정보는 크게 3가지입니다.

1. 한 테이블에 대한 **access method** (index scan, heap scan 등)
2. 테이블의 **join 순서**
3. **join method** (Nested loop join, sort merge join 등)

이러한 정보를 얻기 위해서 모든 가능한 plan의 cost를 측정하고 가장 낮은 cost를 갖는 PLAN을 결정합니다. 이를 위해 parse tree를 query graph 형태로 저장하게 되는데 아래에 매칭되는 주요 용어를 간단하게 정리하였습니다.

- **NODE** : table
- **TERM** : predicate
- **SEGMENT** : column
- **EQCLASS** : join predicate's columns

cost의 계산은 컬럼의 `1 / Number of Distinct Value(NDV)` 을 선택도(selectivity)로 사용합니다. 모든 데이터가 고르게 분포되어 있다고 가정하고 데이터 조회 이전에 몇 개의 행이 조회될지 예측하는 것입니다. 상대적으로 적은 행을 가지는 테이블을 먼저 조회하는 것이 유리할 가능성이 높습니다. 먼저 term(predicate)에 대해서 cost를 계산및 인덱스 스캔이 가능 여부등을 체크합니다. 관련 함수는 `qo_analyze_term()`입니다. term에 대한 분석이 완료되면 node(table)들을 순열하며 생성될 수 있는 모든 경우의 plan의 cost를 비교하고 그중 가장 최적의 plan을 결정합니다. `planner_permutate()`에서 그 작업을 진행합니다.

> OPTIMIZER는 사전에 집계된 통계정보를 바탕으로 EXECUTE PLAN을 생성한다. EXECUTE PLAN에는 JOIN의 순서 및 방법, 그리고 어떠한 INDEX를 선택할 것인가 에 대한 정보가 포함된다. 통계정보에는 컬럼의 selectivity, 전체 row수, 전체 page수가 저장된다. CUBRID에서는 selectivity의 경우 index의 컬럼만 저장되며, 이것은 누적값으로 저장되기 때문에 첫번째 컬럼만 독립적인 selectivity로 활용이 가능하다. join method는 nested loop join과 sort merge join이 있으며, sort merge join은 `USE_MERGE` 힌트를 사용해야 선택되도록 기본 설정으로 정해져 있다. nl-join은 내부적으로 두개로 구분되는데, join 조건을 index에 사용하면 index-join이며, join 조건이 index scan을 하지 않으면 nl-join으로 구분한다.

![optimizer](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/135_optimizer.jpg)

위 그림은 join 순서가 `A->B`이고, index join method가 선택되었을 때 plan 객체를 나태낸 것입니다. 한 박스는 `QO_PLAN` 객체이고 파란색 부분이 `plan_type`, 그 밑의 부분이 공용체인 `plan_un` 정보입니다. parser의 `PT_NODE`와 비슷하게 `plan_type`에 따라 공용체 `plan_un`의 정보가 달라지는 구조입니다.

`PLANTYPE_SCAN`에는 각 테이블의 scan_method와 인덱스 관련 term이 저장됩니다. term(조회조건)은 인덱스 스캔 가능여부에 따라 key range, key filter, data filter로 나눌수 있습니다. 이것이 각각 term, kf term, sarg term에 저장됩니다. `PLANTYPE_JOIN`에는 outer와 inner 테이블 정보 및 join method 정보가 포함되어 있습니다. 그리고 마지막으로 `PLANTYPE_SORT`에 order by 절 관련된 내용이 생성됩니다.

> `PT_NODE` 구조체로 작성된 parse tree를 node, term, segment로 구분하고 최적의 실행계획을 계산하는 OPTIMIZER의 자세한 설명은 아래 링크를 참조하기 바란다.
> http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-optimizer

## XASL GENERATOR

XASL은 **eXtensional Access Specification Language**의 약자입니다. 풀어서 번역해보면 데이터 접근 사양에 관한 언어인데 확장된 형태라는 이야기입니다. CUBRID는 EXCUTOR에서 이 XASL을 해석하여 원하는 데이터를 얻습니다. 접근 사양이라는 것이 결국 어떻게와 어떤 데이터를 접근할 것인가에 대한 사양이고, 이것은 OPTIMIZER에서 생성된 PLAN과 parse tree에 저장되어 있습니다. 이 두가지 정보를 종합하여 XASL 형태로 저장하는 것입니다. XASL generator를 OPTIMIZER 단계의 부분으로 생각해도 무리는 없을 것 같습니다.

![xasl generator](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/137_xasl%20generator.jpg)

위 그림은 JOIN 순서가 `A->B->C`일때 생성된 PLAN을 가지고 `XASL_NODE`가 생성되는 모습을 나타낸것입니다. 한개의 `XASL_NODE`는 한개의 access spec을 가지고 이는 한개의 테이블 조회에 대한 정보를 담는다고 생각하는 것이 이해가 쉬울 것 같습니다. 물론 partition table 같은 경우 한개의 `XASL_NODE`가 여러개의 access spec을 가질 수 있습니다. 일반적으로 SQL에서 table이 존재할 수 있는 위치는 FROM절 입니다. 이것은 join 형태로 작성될 수도 있고, sub query로 작성될 수 있습니다. 위 그림에서는 join으로 연결되어 있으며 이것이 `scan_ptr`로 연결된 것을 확인 할 수 있습니다. 이러한 `XASL_NODE`를 연결하는 주요 포인터는 아래와 같습니다.

- **APTR** : UNCORRELATED SUB QUERY
- **DPTR** : CORRELATED SUB QUERY
- **SCAN_PTR** : JOIN

CORRELATED SUB QUERY는 외부에 참조하는 컬럼이 있는 것이며, UNCORRELATED SUB QUERY는 외부 참조 컬럼이 없는 것을 뜻합니다.

```sql
SELECT 1
   FROM TBL1 A
              ,(SELECT * FROM TBL2 WHERE COL1 = A.COL1) B
              ,(SELECT * FROM TBL3 WHERE COL1 = 3) C
```

위에서 `B`는 CORRELATED SUB QUERY이고, `C`는 UNCORRELATED SUB QUERY입니다. SUB QUERY를 구분한 이유는 CORRELATED SUB QUERY의 경우 참조되는 데이터가 먼저 스캔되야 하기 때문에 SCAN의 순서가 정해지기 때문입니다. 반면에 UNCORRELATED SUB QUERY는 그런 순서의 제약이 없어 바로 scan을 진행 할 수 있습니다. 위 그림은 join만 존재 하기 때문에 `scan_ptr`만 사용되 었는데, sub query가 사용되면 그 성격에 따라 `aptr`과 `dptr`로 XASL node가 연결 됩니다. 거의 대부분의 SQL이 3가지 pointer로 `XASL_NODE`를 연결하는 것으로 작성됩니다.

> XASL관련 자세한 설명은 아래 링크를 참조하기 바란다.
> http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-xasl-generator

## EXECUTOR

XASL을 해석하여 data를 조회하거나, 저장, 변경하는 단계입니다. **Volcano model (Iterator Model)**로 구현 되어 있는데, 이는 데이터를 조회하는데 있어 연관되어 있는 join, sub query, predicate의 평가를 진행하여 하나의 행을 확정하고 다음 행의 조회를 이어서 수행하는 구조입니다. `qexec_execute_mainblock()` 함수가 이 단계의 시작 지점입니다.

![executor](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/138_executor.jpg)

위 그림은 SQL에서 XASL이 생성되고 이를 실행하는 순서를 나타낸 것입니다. 아래와 같은 순서로 scan이 진행됩니다.

1. join의 순서가 첫번째인 `tab a`의 XASL node가 시작 node가 된다.
2. `tab a` node의 `aptr`인 빨간색 박스의 node가 먼저 scan되어 결과를 temp file에 저장한다.
3. `tab a` node에 대한 scan이 진행되고 1row 조회한다.
4. join 관계인 `tab b` node에 대해서 위 단계에서 1row 조회된 데이터를 사용하여 scan을 진행한다.
5. 지금까지 조회된 결과 set을 사용하여 `dptr`인 `tab c` node를 scan한다.
6. 3의 단계로 이동하여 전체 row가 조회될 때까지 반복한다.

`aptr`과 `dptr`과 같은 sub-query는 독립적인 `XASL_NODE`입니다. 그래서 시작지점과 동일한 `qexec_execute_mainblock()` 함수를 호출합니다. 이에 반해 join은 main block과 연결되어 여러 테이블의 스캔을 같이 수행하여 하나의 행을 생성합니다. 호출되는 함수는 `qexec_execute_scan()`입니다. 하나의 행이 결정되어 저장되는 것은 `qexec_end_one_iteration()` 함수에서 진행됩니다.

> executor에 대한 자세한 내용은 아래 링크를 참조하기 바란다.
> http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-executor

지금까지 CUBRID의 Query Process에 대해서 개괄적으로 설명하였습니다. 다음에는 조금 더 자세히 각 단계의 상세 내용에 대해서 이야기 해보겠습니다.
