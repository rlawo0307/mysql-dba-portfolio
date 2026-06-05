# 트랜잭션 메커니즘 (InnoDB)
* InnoDB는 트랜잭션의 ACID 속성과 동시성 제어를 보장하기 위해 아래와 같은 메커니즘을 사용함
    * `Undo Log` : 트랜잭션 rollback 및 MVCC 지원
    * `Redo Log` : 장애 발생 시 데이터 복구
    * `MVCC` : lock 없이 일관된 읽기 제공
    * `Lock` : 트랜잭션 간 동시성 제어
### 전체 흐름(★)
```text
1. update 실행
    * 이전 값 → undo log 기록
    * redo log buffer 기록
    * buffer pool의 데이터 페이지 수정

2. commit
    * redo log buffer → redo log file flush
    * 트랜잭션 commit 처리 완료

3. 이후
    * buffer pool의 dirty page → data file flush

4. purge
    * 더 이상 참조되지 않는 undo log 정리

5. 장애 발생 시(Crash Recovery)
    * Redo Recovery
        * redo log를 재생하여 commit된 변경 내용을 data file에 다시 적용
    * Undo Recovery
        * commit되지 않은 트랜잭션을 undo log를 이용해 rollback
```
<br>

# Undo Log
* 데이터 변경 이전의 값을 저장하는 로그
* 목적
    * 트랜잭션의 [Atomicity](transaction-basics.md)(원자성) 보장 
        * 트랜잭션 수행 중 문제가 발생하거나 rollback 요청 시,
        * 데이터 변경을 취소하고 원래 상태로 되돌리기 위해 undo log를 사용
    * MVCC 지원
        * undo log는 변경 전 데이터 값을 저장
        * 다른 트랜잭션이 과거 snapshot을 읽을 수 있게 함
* 저장 내용 (`Undo Log Entry`)
    * 트랜잭션 ID
    * undo 작업 유형 정보
    * 변경되기 전의 데이터 값
    * undo log의 순서 및 이전 undo log 포인터
    * 원본 레코드 위치 정보(clustered index record)
* 구조
    * 하나의 트랜잭션에서 발생한 여러 undo log는 `체인`(연결 리스트) 형태로 관리됨  
        * 각 트랜잭션이 변경한 데이터에 대한 undo log가 순서대로 연결되어 있음
        * 롤백 시 마지막 변경부터 거꾸로 순서대로 이전 상태로 복구
    * 연결된 여러개의 undo log 들은 `undo log segment` 단위로 관리됨
    * undo log segment는 `undo tablespace` 내에 저장
        * undo tablespace는 일반 데이터 파일과 분리되어 관리됨
* 동작 원리
    * update 수행 시
        1. 변경 전 데이터를 undo log에 기록
        2. 실제 데이터 페이지 변경
        3. 트랜잭션 종료 후
            * rollback 시 : undo log를 적용해 변경 내용을 원래 상태로 복구
            * commit 시 : 변경 내용이 확정됨. undo log는 즉시 삭제되지 않음    
        4. 이후 동작
            * [Consistent Read](consistent-vs-current-read.md) (MVCC)
                * 다른 트랜잭션이 Read View에 따라 undo log를 참조해 과거 버전의 데이터를 조회할 수 있음
            * undo log 정리
                * 참조 중인 Read View가 없을 경우 purge 과정에서 정리됨
<br>

# Redo Log
* 데이터 페이지 변경 내용을 기록하는 로그
* 목적 
    * 트랜잭션의 [Durability](transaction-basics.md)(영속성)을 보장
        * DB 서버가 비정상 종료되었을 경우,
        * 디스크에 반영되지 못한 변경 사항을 다시 재생(redo) 해서 복구
* 저장 내용 (`Redo Log Record`)
    * 페이지 변경 정보
        * tablespace id
        * page id
        * offset
        * 변경 내용
    * 관리 정보
        * redo log 번호(LSN)
        * 트랜잭션 ID
        * checksum 등
* 구조
    * 데이터 페이지 변경 과정에서 `redo log record` 생성
        * 단일 SQL문에서 하나 또는 여러 페이지를 변경할 수 있음
            * 트랜잭션 하나 당 redo log record 하나가 아님
            * SQL문 하나 당 redo log record 하나가 아님
        * 하나의 페이지 변경 과정에서도 여러 redo log record가 생성될 수 있음
    * 여러 redo log record는 `redo log file`에 저장됨
    * redo log file은 순환 방식으로 재사용됨
* 로그 기록 방식
    * InnoDB는 `WAL(Write-Ahead Logging)` 원칙을 따름
        * 변경된 데이터 페이지가 디스크에 반영되기 전에 관련 redo log가 먼저 durable 해야 함
        * redo log 기록 성공 후 commit 허용
            1. 변경 내용(redo log)을 메모리(`redo log buffer`)에 기록
            2. 커밋 시, redo log buffer의 내용을 디스크(`redo log file`)로 flush
            3. 나중에 메모리([buffer pool](../core-concepts/innodb-memory-architecture.md))의 dirty data/index page를 디스크에 반영
                * dirty page : 메모리에만 있고 디스크에는 아직 반영되지 않은 실제 데이터 페이지
                * redo log를 참조하지 않고 메모리의 데이터 페이지를 디스크에 그대로 기록
    * 정상 동작 중에는 복구를 대비해 기록됨
    * 장애 복구 시, 디스크에 저장된 redo log를 재생하여 data file을 복구
<br><br>

# MVCC (Multi-Version Concurrency Control)
* [consistent read](consistent-vs-current-read.md)에서 여러 row 버전을 이용해 lock 없이 읽기 일관성을 제공하는 메커니즘
* 목적
    * 트랜잭션 간 읽기 일관성과 동시성을 보장
        * 현재 row 버전이 snapshot 기준으로 visible하지 않다면,
        * undo log를 따라 과거 row 버전을 읽어 일관된 결과를 반환
        * 각 트랜잭션은 자신의 snapshot 기준으로 visible한 버전을 읽음
* 구성 요소
    * clustered index row
        * 실제 데이터
        * `trx_id` : 이 row를 마지막으로 수정한 트랜잭션 id
        * `roll_pointer` : undo log record를 가리키는 포인터
    * `undo log` : 이전 버전을 저장
    * `read view` : snapshot visibility 판단 기준
* 구조
    * undo log record는 데이터 변경 이전의 값을 저장
    * row가 계속 수정되면 roll_pointer를 통해 `version chain`이 형성됨
```text
Clustered Index (현재 버전)
+-----------------+    
| c1 = 30         |   
| trx_id = 300    |     undo log record 2  
| roll_pointer ───────▶ +-------------+
+-----------------+     | value = 20  |
                        | trx_id=200  |       undo log record 1 
                        | prev ─────────────▶ +-------------+
                        +-------------+        | value = 10  |
                                               | trx_id=100  |
                                               | prev = null |
                                               +-------------+
```
* 동작 원리
    * update 수행 시
        1. 이전 버전의 값을 undo log record에 저장
        2. clustered index의 row 갱신
            * 실제 row 데이터 변경
            * trx_id가 현재 트랜잭션 id로 변경
            * roll_pointer가 생성된 undo log record를 가리키도록 설정
    * select (consistent read) 수행 시 
        1. clustered index에서 현재 row version을 읽음
        2. trx_id 확인
        3. read view와 trx_id를 비교하여 해당 row version의 visibility 판단
            * visible → 현재 row 반환
            * not visible → roll_pointer를 따라 undo log 탐색하며 과거 버전을 찾음
## Read View란?
* 특정 시점의 트랜잭션 상태 정보를 저장하는 InnoDB 내부 metadata 구조
* MySQL MVCC에서는 snapshot read를 위해 read view를 사용
    * snapshot : 특정 시점 기준으로 트랜잭션이 일관되게 읽을 수 있는 데이터 상태
    * read view : snapshot을 구현하기 위한 InnoDB 내부 metadata 구조
    * 즉, 실무적으로 `snapshot 생성 = read view 생성`
* read view를 통해 현재 트랜잭션에서 각 row 버전의 visible 여부를 판단
    * 실제 과거 row 버전은 undo log를 통해 복원
* 생성 시점 [→ read view 생성 시점 및 동작 확인](labs/mvcc-snapshot-check.md)
    * consistent read 시 생성
        * [REPEATABLE READ](isolation-levels.md) : 트랜잭션 내 첫 consistent read 시 생성 후 재사용
        * [READ COMMITTED](isolation-levels.md) : consistent read마다 새로 생성
* 구조
    * `m_ids` : read view 생성 시점에 아직 commit되지 않은 트랜잭션 목록
    * `min_trx_id` : read view 생성 시점 기준, active transaction 중 가장 작은 id 
        * active transaction : 시작 후 아직 종료(commit/rollback)되지 않은 트랜잭션 
    * `max_trx_id` : read view 생성 시점 기준, 다음에 할당될 트랜잭션 id
    * `creator_trx_id` : read view를 만든 트랜잭션
### visibility 판단 기준
* row의 trx_id와 read view를 비교하여 visible 여부를 결정
    * `trx_id = creator_trx_id`
        * 자기 트랜잭션이 변경한 내용
        * visible
    * `trx_id < min_trx_id`
        * read view 생성 전에 이미 commit
        * visible
    * `trx_id >= max_trx_id`
        * read view 생성 이후 시작된 future transaction
        * not visible
    * `trx_id ∈ m_ids`
        * read view 생성 시 active 상태
        * not visible
    * 그 외
        * read view 생성 전에 commit된 것으로 간주
        * visible
