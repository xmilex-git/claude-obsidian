---
type: source
source_type: jira-wiki+pdf
title: "QP Analysis — Parser"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-parser"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - parser
  - bison
  - flex
status: active
related:
  - "[[qp-analysis]]"
  - "[[components/parser]]"
  - "[[components/parser-allocator]]"
  - "[[concepts/PT_NODE]]"
attachments:
  - "_attachments/qp-analysis/parser.jpg"
pdf_sources:
  - ".raw/qp-pdfs/parser_parsing_tree_structure_v1_0.pdf (5p)"
  - ".raw/qp-pdfs/parser_lexing_and_parsing_sql_v1_0.pdf (5p)"
  - ".raw/qp-pdfs/query_processing_cheat_sheet.pdf (5p)"
---

# QP Analysis — Parser

> 원본 wiki: [analysis-for-qp-parser](http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-parser) (PDF 3개 인덱스)
> 본 페이지는 위 3개 PDF + cheat sheet 의 종합.

## High-level API entry points (cheat sheet)

```
/* Client 로부터 */
ux_prepare()
ux_execute()

/* CSQL 로부터 */
csql_execute_statements()
```

메인 컴파일 함수: `db_compile_statement_local()` (in `db_vdb.c`):

```
db_compile_statement_local()
  /* Compilation stage */
  pt_class_pre_fetch()        // schema pre-fetch (3-tier 캐싱)
  pt_compile()                // ── pt_semantic_check() → pt_check_with_info()
  mq_translate()              // view transform + rewrite
  pt_class_pre_fetch()
  /* Preparation stage */
  do_prepare_statement()
```

## PT_NODE 구조

![[_attachments/qp-analysis/parser.jpg]]

**해석** (다이어그램에서 직접):

`SELECT * FROM table a WHERE (col1,col2) = (1,2)` 의 parse tree:

```
PT_SELECT (info.query.q.select)
 ├── list  → PT_VALUE (Type_enum = PT_TYPE_STAR)
 ├── From  → PT_SPEC (info.spec.entity_name → PT_NAME "table a", location=-1)
 └── Where → (rewrite 전: PT_EXPR Op=PT_AND with Arg1/Arg2)
              (rewrite 후: PT_EXPR PT_EQ col1=1 → next → PT_EXPR PT_EQ col2=1)
```

> 우측 노트: "Simple rewrite" — parser 단계에서 `(col1,col2)=(1,2)` 가 `col1=1 AND col2=1` 로 변환되어 들어옴.

**PT_NODE 의 핵심 필드** (분석서 직인용):

| 필드 | 역할 |
|---|---|
| `node_type` (`PT_NODE_TYPE` enum) | PT_SELECT / PT_NAME / PT_EXPR / PT_VALUE / PT_SPEC … |
| `next` | 같은 레벨에서 PT_NODE 연결 (select list, AND-chain, FROM list, …). 링크드 리스트 |
| `or_next` | rewrite 단계 이후 OR-chain. AND 는 `next`, OR 는 `or_next` 로 표현됨 |
| `type_enum`, `data_type` | 값/표현식의 타입 (`PT_TYPE_INTEGER` + precision/scale/collation 등) |
| `info` (`PT_STATEMENT_INFO` union) | node_type 마다 다른 sub-구조체 (`PT_SELECT_INFO` 는 `list/from/where/connect_by/...`, `PT_EXPR_INFO` 는 `op/arg1/arg2/arg3/...`) |
| `data_type` (PT_NODE*) | type_enum 의 도메인 정보 노드 |

> 디버깅 팁 (분석서): gdb 보다 **DDD (display data debugger)** 가 PT_NODE 시각화에 유리.

## Parse tree 순회 메커니즘 — `parser_walk_tree()`

분석서 핵심 인용:

```c
/* preorder 같이 노드 방문 후 순회 */
PT_NODE *parser_walk_tree(PARSER_CONTEXT *, PT_NODE *,
                          PT_NODE_WALK_FUNCTION pre_function, void *pre_argument,
                          PT_NODE_WALK_FUNCTION post_function, void *post_argument);

/* postorder 같이 순회 후 노드 방문 */
PT_NODE *parser_walk_leaves(...);
```

- node_type 마다 sub-tree 진입 순서가 고정. **`pt_apply_f[]` 배열** 이 type → apply function 매핑.
- 예: `pt_apply_select()` 는 `PT_SELECT_INFO` 의 `with → list → from → where → connect_by → start_with → after_cb_filter → group_by → having → using_index → with_increment → order_by → orderby_for → into_list → qcache_hint → check_where → waitsecs_hint → use_merge → index_ls → index_ss → use_idx → use_nl → ordered → for_update → limit` 순서로 자식 노드 호출.
- pre_function 은 `PT_CONTINUE_WALK / PT_LEAF_WALK / PT_LIST_WALK / PT_STOP_WALK` 를 반환하여 자식·or_next·next 의 순회 여부를 통제.

→ [[components/parser]] 의 walker API 와 동일.

## Lexer / Bison build pipeline

분석서 발췌:

```
csql_lexer.l   →  flex   →  lex.yy.c
csql_grammar.y →  bison  →  *.tab.c        ──→ C Compiler → executable
```

CMakeLists.txt 흐름 (커스텀):

```
add_custom_command(OUTPUT csql_grammar.yy
  COMMAND ${CMAKE_COMMAND} -DGRAMMAR_INPUT_FILE=src/parser/csql_grammar.y
                          -DGRAMMAR_OUTPUT_FILE=…/csql_grammar.yy
  MAIN_DEPENDENCY src/parser/csql_grammar.y)
bison_target(csql_grammar … COMPILE_FLAGS "--no-lines --name-prefix=csql_yy -d -r all")
flex_target (csql_lexer   … COMPILE_FLAGS "--noline --never-interactive --prefix=csql_yy")
add_flex_bison_dependency(csql_lexer csql_grammar)
```

- `csql_grammar.y` 가 수정되면 (preprocessor 가) 먼저 `csql_grammar.yy` 를 만들고 bison 으로 `.tab.c` 생성. flex 에는 `csql_lexer.l` 입력.

### Lexer 구조 (csql_lexer.l)

```
%{ Define section %}
%%
Rules section            <어휘규칙> <공백> <C 코드>
%%
User subroutine          (yywrap/csql_yyerror/parser_yyinput 등 재정의)
```

예 (Rules):
```
[sS][eE][lL][eE][cC][tT]    { begin_token(yytext); return SELECT; }
[sS][eE][nN][sS][iI][tT][iI][vV][eE]   { begin_token(yytext); return SENSITIVE; }
[sS][eE][pP][aA][rR][aA][tT][oO][rR]   {
    begin_token(yytext);
    csql_yylval.cptr = pt_makename(yytext);
    return SEPARATOR;
}
```

`YY_USER_ACTION` 매크로가 매 토큰 매치 시 `yybuffer_pos` 를 업데이트하여 location 정보를 보존 (`csql_yylloc.buffer_pos`).

### Grammar 구조 (csql_grammar.y)

```
%{ Prologue %}            (#include "parser.h", YYLTYPE 확장, …)
Bison declarations        %union { int number; bool boolean; PT_NODE *node; char *cptr; container_2/3/4/10; struct json_table_column_behavior jtcb; }
%%
Grammar rules
%%
Epilogue                  (parser_make_func_with_arg_count, parser_make_expression, parser_make_link 등)
```

YYLTYPE 가 `first_line/first_column/last_line/last_column/buffer_pos` 5필드로 확장 — 에러 메시지 와 source location 추적.

PT_NODE 생성 헬퍼:

```c
PT_NODE *parser_make_func_with_arg_count(PARSER_CONTEXT *, FUNC_TYPE, PT_NODE *args_list,
                                         size_t min_args, size_t max_args);
PT_NODE *parser_make_expression(PARSER_CONTEXT *, PT_OP_TYPE OP,
                                PT_NODE *arg1, PT_NODE *arg2, PT_NODE *arg3);
PT_NODE *parser_make_link(PT_NODE *list, PT_NODE *node);  // parser_append_node wrapper
```

→ 새 SQL 구문 추가 시 수정 파일: `src/parser/csql_lexer.l` + `src/parser/csql_grammar.y` (action 부분에 PT_NODE 생성 로직 작성).

## Cross-references

- [[components/parser]] — walker/visitor API + flex/bison wrap
- [[components/parser-allocator]] — PT_NODE arena allocator
- [[components/optimizer-rewriter]] — Parser 가 못 끝낸 rewrite (CNF 등) 다음 단계
- [[qp-analysis-semantic-check]] — `pt_compile()` 이후 단계
- [[concepts/PT_NODE]] — 자료구조 단독 페이지 (있다면)
- 원문/PDF 보존: `.raw/qp-pdfs/parser_*.pdf`, `.raw/qp-analysis-parser.md`
