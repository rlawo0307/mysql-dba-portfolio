# InnoDB 메모리 구조
```text
MySQL Server
 ├─ Global Memory
 │   ├─ Buffer Pool
 │   │   ├─ Page
 │   │   |    ├─ Data Page
 │   │   |    |    ├─ row (data + trx_id + roll_pointer)
 │   │   |    |    ├─ row (data + trx_id + roll_pointer)
 │   │   |    |    └─ ...
 │   │   |    |
 │   │   |    ├─ Index Page
 │   │   |    |    ├─ Root Page
 │   │   |    |    |    └─ Index Entry
 │   │   |    |    |         ├─ key + page pointer
 │   │   |    |    |         ├─ key + page pointer
 │   │   |    |    |         └─ ...
 │   │   |    |    ├─ Internal Page
 │   │   |    |    |    └─ Index Entry
 │   │   |    |    |         ├─ key + page pointer
 │   │   |    |    |         ├─ key + page pointer
 │   │   |    |    |         └─ ...
 │   │   |    |    └─ Leaf Page
 │   │   |    |         └─ Index Entry
 │   │   |    |              ├─ clustered index일 경우, 실제 데이터 row 저장
 │   │   |    |              ├─ Secondary index일 경우, (index key + pk) 저장
 │   │   |    |              └─ ...
 │   │   |    |
 │   │   |    └─ Undo Page
 │   │   |         ├─ old value 1
 │   │   |         ├─ old value 2
 │   │   |         └─ ...
 │   │   | 
 │   │   ├─ Page Management
 │   │   |    ├─ Free List
 │   │   |    ├─ LRU List
 │   │   |    |    ├─ Young 영역 : 자주 사용되는 page
 │   │   |    |    └─ Old 영역 : 새로 읽혔거나 덜 중요한 page
 │   │   |    └─ Flush List
 │   │   |
 │   │   └─ Internal Structure
 │   │        ├─ Change Buffer
 │   │        └─ Adaptive Hash Index (AHI)
 │   │   
 │   └─ Redo Log Buffer
 │
 └─ Session Memory
     ├─ sort buffer
     ├─ join buffer
     ├─ read buffer
     └─ temporary table memory
```
<br>

# Global Memory
## Buffer Pool
* 테이블과 인덱스 데이터를 캐싱하는 메인 메모리 영역
* page 단위로 관리 (default page size: `16K`)
### Page 종류
* `Data Page`
    * 실제 row 데이터가 들어있는 page
    * clustered index의 leaf page에 해당
    * 일반적인 select 수행 시, 최종적으로 접근하는 대상
        * covering index 사용 시에는 접근하지 않을 수 있음
* `Index Page`
    * 인덱스(B+Tree)를 구성하는 page
    * Root Page
        * B+Tree의 최상위 page
        * 모든 탐색의 시작점
        * 항상 1개만 존재
    * Internal Page
        * Root와 Leaf 사이의 중간 page
        * 탐색 경로를 위한 page
        * 0개 이상 존재
    * Leaf Page
        * B+Tree의 최하위 page
        * 모든 탐색은 leaf page에서 종료됨
        * clustered index일 경우, 실제 데이터 저장
            * leaf page = data page
        * secondary index일 경우, (index key + pk) 저장
            * pk를 이용해 clustered index를 다시 탐색(index lookup)
            * 최종적으로 data page에 접근
* `Undo Page`
    * 과거 데이터(이전 버전)를 저장하는 page
        * undo log = 논리적 개념
        * undo page = 물리적 저장 위치
    * 데이터 변경 시,
        * 기존 값은 undo log(undo page)에 저장
        * 새로운 값은 data page에 반영됨 (dirty page 생성)
    * 트랜잭션 rollback 및 MVCC에서 과거 버전을 조회하는데 사용
### 내부 리스트
* `Free List`
    * 아직 사용되지 않은 빈 page 목록
    * buffer pool에 적재할 빈 page가 필요할 경우, free list에서 확보하여 사용
* `LRU List`
    * buffer pool에 올라온 page 중, 어떤 page를 유지하고 제거할지 관리하는 리스트
        * 자주 사용되는 page는 계속 남아있음
        * 오래 사용되지 않은 page는 제거됨
    * MySQL은 단순 LRU가 아닌 scan resistant 구조를 사용
        * 대량 full scan된 page들이 기존 hot page를 전부 밀어내지 않도록 관리
            * young 영역 : 자주 사용되는 page
            * old 영역 : 새로 읽혔거나 덜 중요한 page
* `Flush List`
    * 변경되었지만 아직 disk에 반영되지 않은 page 목록(= dirty page 목록)
    * 데이터 변경 시, dirty page 생성
    * 생성된 dirty page를 flush list에 등록해놓고 나중에 disk로 flush
### Buffer Pool 기반 특수 자료구조
* `Change Buffer`
    * secondary index 변경을 지연 처리하기 위한 구조
    * **buffer pool에 없는 secondary index page**에 대한 변경을 캐싱하는 특수 자료구조
        * 먼저 change buffer에 변경 내용을 저장
        * 나중에 secondary index page가 buffer pool로 올라오면 변경 내용을 merge
    * clustered index에는 적용되지 않음
    * secondary index가 buffer pool에 없을 때만 사용
    * random I/O를 줄이기 위한 write 최적화 구조
    * buffer pool 메모리를 사용하여 동작
        * buffer pool이 부족하면 효과가 떨어짐
* `Adaptive Hash Index(AHI)`
    * 자주 사용되는 인덱스 접근 패턴을 기반으로 내부적으로 생성하는 해시 기반 인덱스
    * 자주 반복되는 index lookup을 hash lookup으로 빠르게 처리
    * buffer pool에 있는 데이터를 기반으로 생성됨
    * 자동 생성/삭제 되며, 사용자가 직접 제어할 수 없음 (기능 on/off는 가능)
    * 충분한 buffer pool 메모리 필요
    * 동시성이 높은 환경에서는 경합으로 인해 병목이 발생할 수 있음
    * 실무에서는 workload에 따라 `innodb_adaptive_hash_index` 옵션을 비활성화하기도 함
## Redo Log Buffer → [트랜잭션 메커니즘](../transaction/transaction-mechanism.md)
* Redo Log File(disk)에 기록되기 전, 변경 내용을 임시로 저장하는 메모리 영역
* 데이터 변경 시, 항상 즉시 disk에 기록하는 것은 성능 저하를 유발함
* 따라서, 메모리에 먼저 기록한 후 commit 시(또는 설정에 따라) disk로 flush하는 전략
    * redo log는 data page보다 먼저 disk에 기록됨([WAL](../transaction/transaction-mechanism.md), Write-Ahead Logging)
    * `commit ≠ 데이터가 디스크에 저장`
    * `commit = redo log를 디스크에 저장`
    * 즉, commit 시 data page는 즉시 disk에 반영되지 않음
        * data page는 이후 비동기적으로 disk에 반영됨
        * redo log를 기반으로 crash 발생 시 복구 및 동기화 가능
## 읽기 동작
```text
SELECT
   ↓
Buffer Pool에 page가 있는지 확인
   ↓ 
- 있으면 → memory(Buffer Pool)에서 읽어옴
- 없으면 → Free List에서 빈 page 확보 → disk에서 page를 읽어 Buffer Pool에 적재
   ↓
결과 반환
```
## 쓰기 동작
```text
INSERT / UPDATE / DELETE
           ↓
필요한 page가 Buffer Pool에 있는지 확인
           ↓
- 있으면 → Buffer Pool에서 바로 읽음 (disk I/O 없음)
- 없으면 → Free List에서 빈 page 확보 → disk에서 page를 읽어 Buffer Pool에 적재
           ↓
Undo Log(Undo Page) 기록
           ↓
Data Page 변경 (dirty page 생성)
           ↓
Redo Log Buffer(memory)에 변경 내용 기록
           ↓
         commit
           ↓
Redo Log File(disk)로 flush
```
## LSN 과 Checkpoint
* 데이터 변경 시, data page는 즉시 반영되지 않고 redo log만 먼저 기록됨
* 즉, data file은 항상 최신 상태가 아닐 수 있음
* 어디까지 disk에 반영됐는지 알 수 있어야 함
### Log Sequence Number(LSN)
* redo log 상의 논리적 위치를 나타내는 값
* 모든 변경 작업은 LSN 기준으로 순서가 관리됨
### checkpoint
* redo log와 data file(disk) 간의 동기화 상태를 나타내는 기준 LSN
    * 해당 LSN까지의 변경 내용은 disk에 반영되었음을 의미
* crash 발생 시, checkpoint 이후의 redo log를 재적용하여 복구
* checkpoint 이전의 redo log는 이미 반영되었으므로 덮어써도 무방함
* redo log 부족 시 동작
    1. dirty page를 disk로 flush
    2. checkpoint를 앞으로 이동
    3. redo log 공간 확보
<br><br>

# Session Memory
* 각 connection(세션)마다 필요 시 동적으로 할당되는 쿼리 실행용 메모리
    * 세션끼리 공유하지 않고 독립적으로 사용
    * 항상 사용하는게 아니라 쿼리 실행 시 연산을 수행할 때 할당
* 세션 수가 많고, 여러 작업이 동시에 수행되면 전체 메모리 사용량이 크게 증가할 수 있음
## Sort Buffer
* order by / group by 처리 과정 중 정렬 작업을 수행하기 위한 세션 메모리
* 인덱스로 정렬을 처리할 수 없으면 filesort 과정에서 사용될 수 있음
* `sort_buffer_size`로 메모리 사이즈 설정 가능
    * 과도하게 크게 설정하면 동시 접속 증가 시 메모리 사용량이 커질 수 있음
## Join Buffer
* join 연산을 수행하기 위한 세션 메모리
* join 중간 결과 및 비교 대상를 저장할 때 사용
* join 조건에 적절한 인덱스를 사용하지 못하거나, 비효율적인 join plan이 선택될 때 사용
* block nested loop join, batched key access 등에서 사용
* `join_buffer_size`로 메모리 사이즈 설정 가능 (default : `256KB`)
    * join buffer가 크면 일부 join에서 I/O를 줄일 수 있음
    * 과도한 사이즈는 메모리 사용량이 커질 수 있음
## Read Buffer
* table scan 시, 데이터 읽기 효율을 높이기 위한 보조 메모리
    * 근본적인 성능 개선책은 아님
    * 인덱스 설계가 더 중요!
* `read_buffer_size`로 메모리 사이즈 설정 가능
## Tmp Table Memory
* 쿼리 실행 중 내부 임시 테이블을 만들 때 사용하는 세션 메모리
* group by, distinct, union, 일부 order by 처리에서 사용
* `tmp_table_size`로 메모리 사이즈 설정 가능
    * default : `16MB`
    * tmp_table_size와 max_heap_table_size 중 작은 값으로 in-memory tmp table 최대 크기를 정의
    * 한도를 초과하면 disk 기반 tmp table로 전환
    * disk tmp table이 많아지면 성능 저하가 발생할 수 있음