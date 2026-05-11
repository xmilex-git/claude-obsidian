---
type: concept
title: "PT_NODE"
status: stub
domain: cubrid-parser
aliases: ["parse tree node", "parser_node"]
created: 2026-05-11
updated: 2026-05-11
tags:
  - concept
  - cubrid
  - parser
  - parse-tree
related:
  - "[[components/parse-tree]]"
  - "[[components/parser]]"
  - "[[sources/qp-analysis-parser]]"
  - "[[Query Processing Pipeline]]"
---

# PT_NODE

CUBRID parser 의 보편 parse-tree node — ~70개 statement/expression type 의 tagged union. 단일 구조체에 union `info` (`PT_STATEMENT_INFO`) 가 node_type 마다 다른 sub-구조체 (`PT_SELECT_INFO`, `PT_NAME_INFO`, `PT_EXPR_INFO`, `PT_VALUE_INFO`, `PT_SPEC_INFO`, `PT_FUNCTION_INFO`, ...) 를 보유. `next`/`or_next` 로 같은 레벨 link.

상세는:
- **컴포넌트 페이지**: [[components/parse-tree]] — 핵심 필드 + type_enum + traversal 전반
- **부모 컴포넌트**: [[components/parser]] — parser_walk_tree / pt_apply_f[] / 메모리 모델
- **분석서**: [[sources/qp-analysis-parser]] — PT_NODE 구조 예시 다이어그램 + parser_walk_tree 의사코드

## 핵심 필드 요약

```c
struct parser_node {
  PT_NODE_TYPE  node_type;     // PT_SELECT / PT_NAME / PT_EXPR / ...
  PT_TYPE_ENUM  type_enum;     // PT_TYPE_INTEGER / PT_TYPE_VARCHAR / ... (semantic 후)
  PT_NODE      *next;          // sibling chain (AND-chain after rewriter)
  PT_NODE      *or_next;       // OR-chain (set by rewriter pt_cnf)
  PT_STATEMENT_INFO info;      // union, node_type-별 활성 멤버
  ...
};
```

## next vs or_next — rewriter 후의 의미

Parser 직후: AND/OR 가 `PT_AND`/`PT_OR` 별도 노드로 표현. Rewriter 의 `pt_cnf()` 후: 같은 레벨 expression 들이 `next` (AND) + `or_next` (OR) 포인터로 평탄화. `PT_AND/PT_OR` 가 살아있는 predicate 는 CNF 변환 실패 (인덱스 스캔 비후보) 신호. 자세히는 [[sources/qp-analysis-rewriter]].

## See also

- [[concepts/QO_ENV]] — Optimizer 가 PT_NODE 트리에서 빌드하는 query graph
- [[concepts/XASL_NODE]] — XASL generator 가 PT_NODE 에서 생성하는 plan
