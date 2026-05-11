---
source: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-resolve-names"
source_type: jira-wiki
title: "analysis-for-qp-semantic-check"
slug: qp-semantic-check
fetched_at: 2026-05-11
captured_via: playwright
language: ko
domain: cubrid-query-processing
stub: true
aliases: [qp-resolve-names]
attachments:
  - id: 149
    filename: "code_analysis_Semantic_check_Overview_v_1_0.pdf"
    url: "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/149_code_analysis_Semantic_check_Overview_v_1_0.pdf"
    local: ".raw/qp-pdfs/semantic_check_overview_v1_0.pdf"
    size_bytes: 224091
  - id: 150
    filename: "code_analysis_Semantic_check-Name_Resolution_v_0_9.pdf"
    url: "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/150_code_analysis_Semantic_check-Name_Resolution_v_0_9.pdf"
    local: ".raw/qp-pdfs/semantic_check_name_resolution_v0_9.pdf"
    size_bytes: 1836660
  - id: 151
    filename: "code_analysis_Semantic_check-Type_Checking_and_Constant_Folding_v_1_0.pdf"
    url: "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/151_code_analysis_Semantic_check-Type_Checking_and_Constant_Folding_v_1_0.pdf"
    local: ".raw/qp-pdfs/semantic_check_type_checking_constant_folding_v1_0.pdf"
    size_bytes: 393052
  - id: 152
    filename: "code_analysis_Semantic_check-Checking_Semantic_of_Particular_Statement_v_0_8.pdf"
    url: "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/152_code_analysis_Semantic_check-Checking_Semantic_of_Particular_Statement_v_0_8.pdf"
    local: ".raw/qp-pdfs/semantic_check_particular_statement_v0_8.pdf"
    size_bytes: 838668
---

# analysis-for-qp-semantic-check (stub — PDF attachments only)

overview : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-

> JIRA 페이지 URL slug는 `analysis-for-qp-resolve-names` 인데 페이지 제목은 `analysis-for-qp-semantic-check` 으로 바뀌어 있다. Name resolution이 semantic check의 한 단계라 slug가 그대로 남은 형태.

본문 없음. 4개 PDF 분석 보고서로 구성된다 (총 ~3.3MB):

- `code_analysis_Semantic_check_Overview_v_1_0.pdf` → 단계 전반
- `code_analysis_Semantic_check-Name_Resolution_v_0_9.pdf` → 이름 해석 (1.8MB, 가장 큰 문서)
- `code_analysis_Semantic_check-Type_Checking_and_Constant_Folding_v_1_0.pdf` → 타입 체크 + 상수 폴딩
- `code_analysis_Semantic_check-Checking_Semantic_of_Particular_Statement_v_0_8.pdf` → 특정 구문(DDL/DML/SELECT 등) 별 의미 체크

각 PDF는 `.raw/qp-pdfs/` 에 다운로드 되었다.
