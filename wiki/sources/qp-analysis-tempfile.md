---
type: source
source_type: jira-wiki
title: "QP Analysis — Temporary File"
source_url: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-tempfile"
parent_source: "[[qp-analysis]]"
created: 2026-05-11
updated: 2026-05-11
language: ko
tags:
  - source
  - cubrid
  - tempfile
  - storage
  - parallel-query
status: active
related:
  - "[[qp-analysis]]"
  - "[[components/query-manager]]"
  - "[[components/query-fetch]]"
  - "[[components/query-executor]]"
attachments:
  - "_attachments/qp-analysis/tempfile_structure.jpg"
  - "_attachments/qp-analysis/list_page.jpg"
  - "_attachments/qp-analysis/numerable.jpg"
  - "_attachments/qp-analysis/temp_file_function.jpg"
---

# QP Analysis — Temporary File

> 원본: [analysis-for-qp-tempfile](http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-tempfile)
> Tempfile 은 QP 단계별 모든 중간 결과를 담는 공통 인프라. 본 페이지 본문 마지막의 "parallel query" 절은 thread-affinity 의 문제점을 직접 다룬다.

## 생성 시점 (분석서 인용)

```
- SELECT 의 결과 저장
- GROUP BY 와 ORDER BY 가 포함된 질의의 정렬 결과 저장
- 부질의 (sub query) 의 결과 저장
- 인덱스 생성 시 정렬 결과 저장
```

## 영구 임시파일 vs 임시 임시파일

| 종류 | 생성 조건 | 제거 시점 |
|---|---|---|
| **영구 임시파일** | 사용자가 직접 볼륨 추가 | 명시적으로 |
| **임시 임시파일** | 영구 임시 공간 소진 시 자동 추가 | CUBRID 서버 종료까지 줄어들지 않음 |

- 시스템 파라메터 `temp_file_max_size_in_pages` — 크기 제한 (기본값: 무제한). 임시파일 생성 시점에 검사하므로 동시 세션이 동시에 만들면 초과 가능.
- `spacedb` 유틸리티로 두 종류 모두 크기 확인 가능.

## LIFE CYCLE

질의 최종 결과 (XASL→list_id) vs 그 외 임시파일이 다르다.

| 임시파일 | LIFE CYCLE | 정리 함수 |
|---|---|---|
| 그 외 (subquery, sort 등) | 질의 종료 시 모두 제거 | `qexec_execute_query` > `qexec_clear_xasl` > `qfile_destroy_list` |
| 질의 최종 결과 | 트랜잭션 단위 (commit/rollback 시 제거) | `log_commit_local` > `file_tempcache_drop_tran_temp_files` |

트랜잭션 단위 관리: 임시파일 생성 시 thread 의 `tran_index` 별로 임시파일 VFID 저장 (`file_Tempcache.tran_files`). commit 시 이 entry 들을 일괄 삭제.

**예외**: `HOLDABLE CURSOR` 와 `QUERY RESULT CACHE` 는 commit 이후로 보존 연장 (`file_temp_retire_preserved`). 본 분석서는 더 다루지 않음.

## 메모리 버퍼

생성 시 최초 몇 개 page 는 데이터 버퍼가 아닌 메모리에서 직접 할당 → 성능 향상. `temp_file_memory_size_in_pages` 로 page 개수 조절.

```
qmgr_create_new_temp_file       // 메모리 버퍼만 할당, 실제 파일 미생성
  ↓ 쓰기 누적
qmgr_get_new_page                // 메모리 부족 시
  → file_create_temp             // 실제 임시파일 생성
```

생성 후 새 page 는 데이터 버퍼에서 할당.

## 임시파일 캐시 (전역 풀)

생성/제거 비용이 큰 임시파일을 캐시에 보관 후 재사용. `max_entries_in_temp_file_cache`, `max_pages_in_temp_file_cache`.

**Trade-off** (분석서 본문 직인용):

> 캐시된 임시파일이 소유한 page 는 공유가 불가하므로, 공간을 사용하는데 있어서 비효율적이다. 예를 들어 영구 임시볼륨이 512M 이어도 이미 캐시된 page 들은 공유가 불가하기 때문에 512M 가 필요한 임시파일은 생성이 불가하고 임시 임시 파일이 생성될 것이다. 이는 공간을 비효율적으로 사용하는 것이지만, 오히려 자원을 독점하지 않도록 방어하는 기능을 수행하기도 한다.

캐시 API:

| 함수 | 동작 |
|---|---|
| `file_tempcache_put` | 캐시로 임시파일 반환 |
| `file_tempcache_get` | 캐시에서 임시파일 획득 |
| `file_tempcache_push_tran_file` | 임시파일 VPID 를 트랜잭션별 정보에 저장 (commit/rollback 시 삭제 위해) |
| `file_tempcache_pop_tran_file` | 트랜잭션별 정보에서 제거 |

## 임시파일 구조

![[_attachments/qp-analysis/tempfile_structure.jpg]]

**해석**: page 들의 링크드 리스트.

```
[prev | ovfl | next]    [prev | ovfl | next]
 NULL ←┴ next  → ↔ prev ←┴ next →  NULL
   page header (data 1, data 2 ...)
   ovfl → 별도 overflow chain (data → data → NULL)
```

- 첫 page 의 `prev = NULL_page_id`, 마지막 page 의 `next = NULL_page_id`.
- 페이지 크기를 초과하는 큰 레코드는 **overflow page chain** 으로 분리. overflow 정보는 레코드 헤더가 아닌 **page 헤더** 에 보관 — 한 page 에 하나의 큰 레코드만 overflow 가능 (CUBRID 의 의식적 설계: 순차 읽기 가정으로 단순화/공간 효율 우선).

## List page vs Slotted page

![[_attachments/qp-analysis/list_page.jpg]]

**해석**: 32-byte PAGE HEADER 동일. **List page** 는 record 들이 위에서 아래로 sequential pack (aaa/bbbb/cc/ddddd). **Slotted page** 는 위로 데이터 + 아래로 slot directory (offset=42 length=6, offset=40 length=2, offset=35 length=5, offset=32 length=3, `is_del=0`).

| List page | Slotted page |
|---|---|
| random access 불가 (sequential only) | random access |
| record 삭제/추가 없음 | 삭제/추가 가능 |
| slot 정보 불필요 — record header 의 길이로 다음 offset 계산 | slot table 필요 |
| 단순 + 공간 효율 | 유연성 |

→ 임시파일이 **순차 쓰기/순차 읽기** 에만 사용되는 조건이 단순한 구현을 가능케 함.

## NUMERABLE 임시파일

![[_attachments/qp-analysis/numerable.jpg]]

**해석**: 일반 임시파일 + "N번째 page 의 VPID 를 가져오는 기능".

- 일반 임시파일에서 N번째 page 를 찾으려면 처음부터 N번 fix 해야 함 (링크드 리스트). 비효율.
- Numerable 임시파일은 **File Header Page** 에 **User Pages Table** 추가 — 임시파일 소유 VPID 들을 array 로 저장.
- 한 page 에 다 못 담으면 extension page chain: `FILE_EXTDATA_HEADER (vpid_next, max_size, size_of_item, n_items) + VPID array`.

용도: **External sort** 의 다중 run 저장 (각 run 의 시작 page 메타로 빠른 random access), **Extendible hash**.

## EXECUTOR 에서 사용 — open/close 패턴

![[_attachments/qp-analysis/temp_file_function.jpg]]

**해석**: `QFILE_LIST_ID` (First_vpid, Last_vpid, last_pgptr) 가 메타. FIX/UNFIX 화살표가 page 전진을 표현. file_alloc() 가 새 page 필요 시 호출, 마지막엔 NULL_page_id 도달.

> 임시파일은 순차로 쓰는 특성. 필요한 page 를 fix 하고 데이터 전부 입력 후 unfix. 여러 세션이 동시에 읽기/쓰기 하지 않으므로 fix/unfix 횟수를 줄여 효율 향상.

### 쓰기 API

```c
qfile_open_list();          // 메모리 버퍼 할당, 임시파일 생성 지연
                            // (qmgr_create_new_temp_file)
qfile_generate_tuple_into_list();
                            // page 에 데이터 넣음, 부족하면 새 page 할당 +
                            // 이전 page unfix (qmgr_get_new_page)
qfile_close_list();         // 마지막 page unfix
qfile_destroy_list();       // 메모리 버퍼 반환 + 임시파일 캐시 or 제거
```

### 읽기 API

```c
qfile_open_list_scan();     // 첫 page fix, current offset 초기화
qfile_scan_list_next();     // 다음 레코드 읽기 + 필요시 다음 page fix + 이전 unfix
qfile_close_scan();         // 마지막 page unfix
```

## 주요 함수 카탈로그

### 메모리 버퍼 사용

| 함수 | 역할 |
|---|---|
| `qmgr_create_new_temp_file` | 임시파일 미생성, 메모리버퍼 캐시 공간만 할당 |
| `qmgr_get_new_page` | 새 page 필요 시 호출. 메모리 다 차면 임시파일 생성 |
| `qmgr_free_list_temp_file` | 메모리 버퍼만 사용된 경우 메모리 해제. 임시파일 있으면 제거 |
| `qmgr_get_old_page` | 기존 파일 page 읽기. 메모리 버퍼면 메모리 주소, page 면 fix 후 page 주소 |
| `qmgr_create_result_file` | result cache 의 최종 결과. 메모리 버퍼 캐시 우회하고 바로 임시파일 생성 (임시 버퍼 장기 점유 방지 추정) |

### 임시파일 생성

| 함수 | 역할 |
|---|---|
| `file_create_temp` | 캐시에 가용 자원 있으면 그것 반환. 없으면 실제 생성 |
| `file_temp_retire` | 임시파일이 1000 page 이상 또는 캐시된 개수 512 이상이면 제거. 그렇지 않으면 캐시 |
| `file_temp_retire_preserved` | result cache / holdable cursor 용. commit 이후에도 유지하기 위해 tran_id 별 entry 가 이미 제거된 상태 → entry 생성 후 캐시 |
| `file_destroy` | 실제 임시파일 제거 |

## Parallel query 영향 (분석서 본문)

CUBRID thread 가 사용하는 자원: **데이터 버퍼의 PAGE, 임시파일, private 메모리 동적 할당**. 병렬 질의에서는:

> 자원을 완전히 독립적으로 구현하거나, 자원이 Thread 정보와 상관없이 사용되어야 한다.

### Temporary file

- 생성/소멸은 thread 간 공유 가능.
- 그러나 **`file_Tempcache.tran_files`** 는 thread-local — commit 시 임시파일을 제거하기 위해 저장. 다른 thread 에 동일 `tran_id` 를 쓰면 pop/push 연산에 락 필요.
- **External sort** 의 경우 생성+사용 후 즉시 제거하므로 트랜잭션 관련 정보 저장 불필요. **preserve 임시파일** 패턴처럼 최초 생성 시 트랜잭션별 정보를 입력 안 하면 됨.

### Page

- FIX 시 **thread 기준 (어떠한 기준인지 확인 필요)** 으로 page id 를 저장. 중간 abort 시 UNFIX 를 위함으로 추정 (확인 필요).
- 그러므로 FIX 와 UNFIX 는 동일 thread 에서.
- `qfile_open_list()` / `qfile_close_list()` 에서 발생했던 문제는 임시파일 문제가 아니고 마지막 page 의 UNFIX 시 저장된 정보 부재 — fix/unfix 자체는 thread 간 독립적으로 구현 가능해 보임.

### Private memory allocation

- `malloc` / `free` 는 문제 없음.
- `private_alloc` / `private_free` 는 thread 독립 필요. 각 thread 가 `private_heap_id` 에 고유 메타정보 주소 보유.
- thread 간 정보가 다르므로 특정 thread 에서 할당된 동적 메모리는 동일 thread 에서 free.
- 우회: `db_change_private_heap(thread_p, 0)` → heap_id 0 으로 설정 시 일반 `malloc/free` 폴백.

→ 이 절은 [[components/parallel-query]] 류의 후속 작업이 참조해야 할 인프라 제약.

## Cross-references

- [[components/query-manager]] — `qmgr_*` (메모리 버퍼 + temp file 게이트웨이)
- [[components/query-fetch]] — `qfile_*` API
- [[components/query-executor]] — pages 의 실 사용 소비자
- [[qp-analysis-executor]] — group by/sort 가 모두 temp file 사용
- 원문 한국어 본문 보존: `.raw/qp-analysis-tempfile.md`
