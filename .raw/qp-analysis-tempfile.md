---
source: "http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-qp-tempfile"
source_type: jira-wiki
title: "analysis-for-qp-tempfile"
slug: qp-tempfile
fetched_at: 2026-05-11
captured_via: playwright (mcp__playwright__playwright_navigate + evaluate)
language: ko
domain: cubrid-query-processing
attachments:
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/166_Structure%20of%20temporary%20file.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/167_list%20page.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/168_Numerable.jpg"
  - "http://jira.cubrid.com:8888/plugins/servlet/page-attachment/169_temp%20file%20function.jpg"
---

# analysis-for-qp-tempfile

overview : http://jira.cubrid.com:8888/wiki/p/RND/view/analysis-for-query-processing-

## 임시 파일 (TEMPORARY FILE)

임시 파일은 질의를 실행할때 발생되는 모든 결과를 저장하며, 아래와 같은 경우에 생성된다.

- SELECT의 결과 저장
- GROUP BY와 ORDER BY가 포함된 질의의 정렬결과 저장
- 부질의의 결과 저장
- 인덱스 생성시 정렬 결과 저장

임시파일은 영구 임시파일과 임시 임시파일로 구분된다. 영구 임시파일은 사용자가 직접 볼륨을 추가한 경우이다. 영구 임시 공간을 전부 사용했을 때 CUBRID는 일시적 임시볼륨을 추가하며, `temp_file_max_size_in_pages` 파라메터로 이 크기를 제한할 수 있다. default 값은 무제한이며, 이 제한은 임시파일이 생성되는 시점에 이루어지므로, 여러 세션에서 동시에 임시파일이 생성되면 제한을 넘을 수 있다. 일시적 임시파일은 CUBRID 서버가 종료되기 전까지 줄어들지 않는다. `spacedb` 유틸리티를 사용하여 영구 임시볼륨과 일시적 임시볼륨의 크기를 확인 할 수 있다.

## 임시파일의 LIFE CYCLE

임시파일의 LIFE CYCLE은 질의 최종 결과(XASL->list_id)에 대한 임시파일과 그 외의 임시파일이 다르다. 부질의의 결과나 정렬의 결과처럼 질의의 최종 결과가 아닌 경우, 최종 결과가 생성되면 필요가 없어지며, 질의가 종료되는 시점에 모두 제거된다. (`qexec_execute_query` > `qexec_clear_xasl` > `qfile_destroy_list`)

질의의 최종 결과인 경우 트래잭션 단위로 관리 되며, commit과 rollback시 제거된다. 이를 위해서 임시파일 생성시 thread의 tran_index별로 임시파일 VFID를 저장하며(`file_Tempcache.tran_files`), 이후 이 정보를 사용하여 트랜잭션에서 사용한 임시파일을 전부 삭제한다. (`log_commit_local` > `file_tempcache_drop_tran_temp_files`)

HOLDABLE CURSOR와 QUERY RESULT CACHE는 질의 최종 결과의 임시파일 제거를 commit 이후로 연장 할 수 있다. 여기서는 이에 대한 분석을 더 진행하지 않는다.

## 임시파일의 메모리 버퍼

임시파일 생성시 최초 몇개의 page는 메모리 버퍼를 사용한다. 데이터 버퍼의 page를 사용하지 않고, 메모리를 직접 사용하므로 성능 향상을 기대할 수 있다. `temp_file_memory_size_in_pages` 시스템 파라메터로 메모리 버퍼 page 개수를 설정할 수 있다. 임시파일 생성시(`qmgr_create_new_temp_file`) 실제 임시파일을 생성하지 않고 메모리버퍼의 page를 할당한다. 임시파일의 쓰기 작업이 계속 진행되어 새로운 page가 필요하고 메모리 버퍼 page가 더이상 없을 경우에 실제 임시파일을 생성한다(`qmgr_get_new_page` > `file_create_temp`). 임시 파일이 생성된 이후에는 새로운 page를 데이터 버퍼에서 할당받는다.

## 임시파일 캐시

임시파일을 생성하고 제거하는 작업은 부하가 큰 작업이다. (`file_create`, `file_destroy`) 생성된 임시파일을 제거하지 않고 캐시에 저장하여 재사용함으로써 성능 향상을 기대할 수 있다. 시스템 파라메터 `max_entries_in_temp_file_cache`, `max_pages_in_temp_file_cache`를 통해서 템프파일 캐시개수와 최대크기를 설정할 수 있다. 캐시된 임시파일이 소유한 page는 공유가 불가하므로, 공간을 사용하는데 있어서 비효율적이다. 예를 들어 영구 임시볼륨이 512M이어도 이미 캐시된 page들은 공유가 불가하기 때문에 512M가 필요한 임시파일은 생성이 불가하고 임시 임시 파일이 생성될 것이다. 이는 공간을 비효율적으로 사용하는 것이지만, 오히려 자원을 독점하지 않도록 방어하는 기능을 수행하기도 한다.

임시파일 캐시와 관련된 함수는 아래와 같다.

- `file_tempcache_put` : 전역 템프 캐시로 임시파일을 이동한다. 임시파일의 사용이 끝나 반환할 때 사용한다.
- `file_tempcache_get` : 전역 템프 캐시에서 임시파일을 가져온다. 임시파일이 필요한경우 사용한다.
- `file_tempcache_push_tran_file` : 임시파일 VPID를 트랙잰션별 정보에 저장한다. 임시파일이 필요한 경우 임시파일을 할당받고 트랜잭션별 정보에 저장한다. 이는 commit, rollback시 임시파일을 삭제하기 위함이다.
- `file_tempcache_pop_tran_file` : 임시파일 VPID를 트랜잭션별 정보에서 제거한다. 임시파일 사용이 끝나 제거시에 사용된다.

## 임시파일 구조

![Structure of temporary file](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/166_Structure%20of%20temporary%20file.jpg)

임시파일은 page들의 모음입니다. 첫번째 page가 파일ID(VFID)이고, 각 page는 링크드 리스트 형태로 연결되어 있습니다. page보다 큰 레코드를 담기 위해서 overflow page를 사용하고, 이것은 page 헤더에 저장됩니다. 레코드 헤더에 저장하지 않고 page 헤더에 저장하는 할 수 있는 것은 큰 레코드가 page의 남은 공간을 전부 저장하고 나머지는 overflow page에 저장하기 때문입니다. 즉 한개의 page에는 한개의 레코드의 overflow page밖에 존재 할 수 없습니다. 임시파일은 순차로 읽기 때문에 로직을 단순화하고 공간 활용성을 최대화하고 있습니다. 예를 들어 레코드에 overflow 여부 헤더를 추가하면, 레코드마다 overflow 정보를 가질수 있지만 순차읽기시 필요없는 기능입니다. 또한 추가된 레코드 헤더만큼 공간 활용성이 줄어듭니다.

![list page](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/167_list%20page.jpg)

list page는 slotted page보다 단순하게 구현되어 있습니다. random access가 없고, 레코드의 삭제와 추가가 없기 때문에 slot정보가 필요없으며, 레코드 헤더의 레코드 길의를 통해서 다음 레코드의 offset을 알 수 있습니다.

## NUMERABLE 임시파일

![Numerable](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/168_Numerable.jpg)

External sort와 Extendible hash에서 NUMERABLE 임시파일을 사용합니다. 일반 임시파일보다 추가된 기능은 몇번째 page의 VPID를 가져오는 기능입니다. 임시파일도 page별로 링크드 리스트로 연결되어 있기 때문에 몇번째 page를 가져오는 기능 구현이 가능합니다. 그러나 N번째까지 전체 page를 fix해야 알 수 있기 때문에 비효율이 발생합니다. Numerable 임시파일은 파일 헤더에 'user pages table'정보가 추가됩니다. 이 정보는 임시파일이 소유한 VPID가 저장되며, 이 정보를 활용하여 적은 page만 fix하여 몇번째 VPID를 가져오는 것 이 가능합니다. External sort의 경우 여러개의 run들을 한개의 임시파일에 저장하고 각 run의 저장된 page 순번을 메타정보로 저장합니다. 몇번째 page로 찾아가는 것이 필요하기 때문에 Numerable 임시파일을 사용하고 있습니다.

## 임시파일 주요 함수

임시파일 생성시 모두 메모리 버퍼를 사용하는 것은 아니다. External sort에서는 실제 파일을 생성하는 함수를 호출하고 있으며, Executor의 스캔 과정에서는 메모리버퍼를 사용하는 함수를 호출하고 있다. 메모리 버퍼를 사용하는 로직이 추가되었을 뿐 결국 두 함수는 파일 생성이라는 동일한 로직을 사용한다.

**메모리 버퍼 사용 함수**

- `qmgr_create_new_temp_file` : 임시파일을 생성하지 않고, 임시 메모리버퍼 캐시 공간을 할당한다.
- `qmgr_get_new_page` : 파일 생성후 새로운 page가 필요할 때 호출한다. 메모리 버퍼 캐시 공간을 다 할당한 경우 임시파일을 생성한다.
- `qmgr_free_list_temp_file` : 메모리 버퍼만 사용하고 실제 파일이 생성되지 않았을 경우에는 메모리 버퍼만 할당 해제한다. 임시파일이 생성되었을 경우에는 제거한다.
- `qmgr_get_old_page` : 이미 생성된 파일의 page를 읽기위해 사용한다. 메모리 버퍼일 경우 해당 메모리 주소를 반환하며, page인 경우 fix하여 page 주소를 반환한다.
- `qmgr_create_result_file` : result cache일때 최종결과 임시파일을 생성할때 사용한다. `qmgr_create_new_temp_file()` 와 다른점은 메모리 버퍼 캐시를 사용하지 않고 바로 임시파일을 생성한다. 임시 메모리버퍼가 오래 할당되는 것을 막기 위함으로 보인다.

**임시파일 생성**

- `file_create_temp` : 임시파일 캐시에 가용한 자원이 있으면 그 임시파일을 리턴한다. 가용한 자원이 없을 경우 실제 임시파일을 생성한다.
- `file_temp_retire` : 임시파일이 1000 page 이상이거나 캐시된 개수가 512개 이상일 경우 임시파일을 제거한다. 그렇지 않으면 임시파일 자원을 캐시하여 재활용합니다.
- `file_temp_retire_preserved` : result cache나 holdable cursor의 경우 preserved가 설정되며, commit이후에도 임시파일을 유지하기 위해 tran_id별로 저장된 임시파일 entry가 제거된 상태이다. 그렇기 때문에 entry를 생성하여 임시파일 자원을 캐시한다.
- `file_destroy` : 실제 임시파일을 제거한다.

## EXECUTOR에서 임시파일의 사용

![temp file function](http://jira.cubrid.com:8888/plugins/servlet/page-attachment/169_temp%20file%20function.jpg)

EXECUTOR에서는 임시파일을 open하고 close하는 형태로 작성되어 있으며, 읽기와 쓰기가 구분되어 있습니다.

임시파일은 순착적으로 쓰는 특성이 있습니다. 그래서 필요한 page를 fix하고 데이터를 전부 입력한 이후에 해당 page를 unfix합니다. 이는 여러 세션이 임시파일을 동시에 읽거나 쓰지 않기 때문에 가능하며, fix와 unfix를 줄여 더 효과적으로 쓰고 읽기가 가능합니다.

**쓰기**

- `qfile_open_list` : 임시 파일의 생성을 미루고 임시 메모리 버퍼를 할당합니다. (`qmgr_create_new_temp_file`)
- `qfile_generate_tuple_into_list` : page에 데이터를 넣을 수 있으면 넣고, 부족하면 새로운 page를 할당받아 데이터를 넣습니다. page를 할당 받을 때 이전 page를 unfix하고 새로운 page를 fix합니다. 메모리버퍼 관련 함수 `qmgr_get_new_page()` 가 호출되기 때문에 메모리 버퍼 사용후 임시파일을 생성합니다. 임시파일 역시 캐시된 임시파일을 사용합니다.
- `qfile_close_list` : 마지막 page를 unfix합니다. 이것은 앞서 page를 할당 받을 때(`qmgr_get_new_page`) fix한 page가 쓰기 작업중에는 unfix하지 않기 때문입니다.
- `qfile_destroy_list` : 메모리 버퍼를 반환하고 임시파일을 캐시합니다. 캐시 공간이 부족하거나 파일 page가 클경우 제거합니다.

**읽기**

- `qfile_open_list_scan ()` : 임시파일의 첫번째 page를 fix하고, current offset을 초기화 합니다.
- `qfile_scan_list_next ()` : 최근 offset을 사용하여 다음 레코드를 읽습니다. 필요한 경우 다음 page를 fix하고 이전 page를 unfix합니다.
- `qfile_close_scan ()` : 최근 page를 unfix합니다.

---

## parallel query

CUBRID의 thread가 동작하면서 사용하는 자원은 데이터 버퍼의 PAGE, 임시파일, private 메모리 동적 할당이 있다. 병렬질의를 수행하기 위해서는 Thread간 자원을 완전히 독립적으로 구현하거나, 자원이 Thread 정보와 상관없이 사용되어야 한다. 예를 들어 thread 1에서 생성한 임시파일이나 메모리 동적할당을 thread 2에서 제거할 수 있어야 한다.

트랜잭션 index를 부모와 동일하게 가져가야 하므로 그 와 관련된 로직을 찾아 수정해야한다. 임시파일의 트랜잭션별 캐시 정보가 이에 해당한다.

**Temporary file**
임시파일 캐시나 result 캐시 로직을 보았을 때 임시파일의 생성과 소멸은 thread간 공유가 가능하다. 그러나 commit시 임시파일을 제거하기 위해 저장하는 정보(`file_Tempcache.tran_files`)가 문제가 된다. 이를 위해 다른 thread에 동일한 tran_id를 사용하는 것은 해당정보의 pop과 push와 같은 연산시 락을 필요로 한다. external sort의 경우 임시파일을 생성 및 사용후 제거하므로 트랜젝션 관련 정보를 저장할 필요가 없다. preserve 임시파일과 비슷하게 최초 생성시 트랜잭션별 정보를 입력하지 않으면 된다.

**Page**
FIX시 thread기준(어떠한 기준인지 확인 필요) 으로 해당 page id를 저장한다. 이는 중간에 abort시 UNFIX를 하기 위함으로 보인다.(확인필요) 그러므로 관련해서 수정이 없다면 FIX와 UNFIX는 동일한 thread에서 진행되어야 한다. `qfile_open_list()`와 `qfile_close_list()`에서 관련 문제가 발생하였지만, 이는 임시파일의 문제가 아니고 마지막 page를 UNFIX하는 과정에서 저장된 정보가 없어서 발생하는 문제이다. (지금까지는 FIX와 UNFIX는 thread간 독립적으로 구현이 가능한 것으로 보인다.)

**private memory allocation**
malloc과 free는 문제가 되지 않는다. `private_alloc`과 `private_free`는 thread 독립적으로 수행되어야 한다. thread는 `private_heap_id`에 고유의 동작할당을 위한 메타정보의 주소정보를 저장한다. thread 간에 이 정보가 다르기 때문에 특정 thread에서 할당된 동적 메모리는 동일한 thread에서 free 되어야한다. 이를 피하기 위해서 `db_change_private_heap (thread_p, 0)`를 통해 0으로 설정하는 경우가 있는데 이런 경우 일반 malloc과 free가 수행된다.
