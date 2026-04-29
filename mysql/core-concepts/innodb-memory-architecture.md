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
<br><br>

# Session Memory
### 개념
* 각 connection 별로 사용하는 메모리
* 세션 단위 메모리
### 구성
* sort buffer
* join buffer
* read buffer
* tmp table memory
### 특징
* connection 수 만큼 증가
* 과도하면 OOM 위험
