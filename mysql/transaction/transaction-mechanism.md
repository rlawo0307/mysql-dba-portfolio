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

3. 이후
    * buffer pool의 dirty page → data file flush
    * undo log → MVCC에서 사용 가능

4. purge
    * 더 이상 참조되지 않는 undo log 정리

5. 장애 발생 시(Crash Recovery)
    * Redo Recovery
        * redo log를 재생하여 commit된 변경 내용을 data file에 다시 적용
    * Undo Recovery
        * commit되지 않은 트랜잭션을 undo log를 이용해 rollback
```
## Undo Log
* 데이터 변경 이전의 값을 저장하는 로그
### 목적
* 트랜잭션의 `Atomicity(원자성)` 보장 [→ 트랜잭션 속성(ACID)에 대한 내용 자세히 보기](transaction-basics.md)
    * 트랜잭션 수행 중 문제가 발생하거나 rollback 요청 시,
    * 데이터 변경을 취소하고 원래 상태로 되돌리기 위해 undo log를 사용
* MVCC 지원
    * undo log를 변경 전 데이터 값을 저장하고 있음
    * 다른 트랜잭션이 과거 snapshot을 읽을 수 있게 함
### 저장 내용 (`Undo Log Entry`)
* 트랜잭션 ID
* undo log 타입
    * INSERT Undo
    * DELETE Undo
    * UPDATE Undo
    * DDL Undo
    * TRUNCATE Undo
* 변경되기 전의 데이터 값
* undo log의 순서 및 이전 undo log 포인터
* 원본 레코드 위치 정보(clustered index record)
### 구조
* 하나의 트랜잭션에서 발생한 여러 undo log는 `체인`(연결 리스트) 형태로 관리됨  
    * 각 트랜잭션이 변경한 데이터에 대한 undo log가 순서대로 연결되어 있음
    * 롤백 시 마지막 변경부터 거꾸로 순서대로 이전 상태로 복구
* 연결된 여러개의 undo log 들은 `undo log segment` 단위로 관리됨
* undo log segment는 `undo tablespace` 내에 저장
    * undo tablespace는 일반 데이터 파일과 분리되어 관리됨
### 동작 원리
* update 수행 시
    1. 변경 전 데이터를 undo log에 기록
    2. 실제 데이터 페이지 변경
    3. 트랜잭션 종료 후
        * rollback 시 : undo log를 적용해 변경 내용을 원래 상태로 복구
        * commit 시 : 변경 내용이 확정됨. undo log는 즉시 삭제되지 않음    
    4. 이후 동작
        * Consistent Read(MVCC)
            * 다른 트랜잭션이 Read View에 따라 undo log를 참조해 과거 버전의 데이터를 조회할 수 있음
        * undo log 정리
            * 참조 중인 Read View가 없을 경우 purge 과정에서 정리됨
## Redo Log
* 데이터 변경 내용을 기록하는 로그
### 목적 
* 트랜잭션의 `Durability(영속성)`을 보장 [→ 트랜잭션 속성(ACID)에 대한 내용 자세히 보기](transaction-basics.md)
    * DB 서버가 비정상 종료되었을 경우, 디스크에 반영되지 못한 변경 사항을 다시 재생(redo) 해서 복구
### 저장 내용 (`Redo Log Record`)
* 페이지 변경 정보
    * tablespace id
    * page id
    * offset
    * 변경 내용
* 관리 정보
    * redo log 번호(LSN)
    * 트랜잭션 ID
    * checksum 등
### 구조
* 데이터 페이지 변경 과정에서 `redo log record` 생성
    * 단일 SQL문에서 하나 또는 여러 페이지를 변경할 수 있음
        * 트랜잭션 하나 당 redo log record 하나가 아님
        * SQL문 하나 당 redo log record 하나가 아님
    * 하나의 페이지 변경 과정에서도 여러 redo log record가 생성될 수 있음
* redo log record는 순환 구조로 저장됨
### 로그 기록 방식
* InnoDB는 `WAL(Write-Ahead Logging)` 원칙을 따름
    * 데이터 페이지를 디스크에 쓰기 전에 반드시 redo log를 먼저 디스크에 기록
        1. 변경 내용(redo log)을 메모리(`redo log buffer`)에 기록
        2. 커밋 시, redo log buffer의 내용을 디스크(`redo log file`)로 flush
        3. 나중에 메모리(`buffer pool`)의 dirty page를 디스크(`data file`)에 반영
            * dirty page : 메모리에만 있고 디스크에는 아직 반영되지 않은 실제 데이터 페이지
            * redo log를 참조하지 않고 메모리의 데이터 페이지를 디스크에 그대로 기록
* 정상 동작 시, redo log는 기록되기만 할 뿐 사용되지는 않음
* 장애 복구 시, 디스크에 저장된 redo log를 재생하여 data file을 복구
## MVCC(Multi-Version Concurrency Control)
### 목적
* lock 없이 동시성을 높이기 위한 읽기 메커니즘
### 핵심 아이디어
* **데이터의 여러 버전을 유지하는 것**
* row의 이전 버전을 undo log에 유지하여 트랜잭션이 snapshot 기준으로 적절한 row version을 읽을 수 있도록 함
* 각 트랜잭션은 자신의 snapshot(=`Read View`)을 읽음 [→ MVCC snapshot 실습](labs/mvcc-snapshot-check.md)
    * read view는 snapshot 시점의 `트랜잭션 상태 정보`를 저장
    * read view는 트랜잭션이 처음 데이터에 접근할 때(select/update 등) 생성됨
### 저장 내용 및 구조
* Clustered Index leaf node 구조
    * 실제 row 데이터
    * `trx_id` : 이 row를 마지막으로 수정한 트랜잭션 id
    * `roll_pointer` : undo log record를 가리키는 포인터
        * undo log record는 데이터 변경 이전의 값을 저장
        * row가 계속 수정되면 roll_pointer를 통해 `version chain`이 형성됨
```text
Clustered Index (현재 버전)
+-----------------+    
| c1 = 30         |   
| trx_id = 300    |          undo2  
| roll_pointer ───────▶ +-------------+
+-----------------+     | value = 20  |
                        | trx_id=200  |            undo1 
                        | prev ─────────────▶ +-------------+
                        +-------------+        | value = 10  |
                                               | trx_id=100  |
                                               | prev = null |
                                               +-------------+
```
* Read View 구조
    * `m_ids` : snapshot 생성 시점에 아직 commit되지 않은 트랜잭션 목록
    * `min_trx_id` : active trx 중 가장 작은 id
    * `max_trx_id` : 다음에 할당될 트랜잭션 id
    * `creator_trx_id` : snapshot을 만든 트랜잭션
### 동작 원리
* update 수행 시
    1. 이전 버전의 값을 undo log record에 저장
    2. clustered index의 row 갱신
        * 실제 row 데이터 변경
        * trx_id가 현재 트랜잭션 id로 변경
        * roll_pointer가 생성된 undo log record를 가리키도록 설정
* select 수행 시 
    1. clustered index에서 현재 row version을 읽음
    2. trx_id 확인
    3. read view와 trx_id를 비교하여 해당 row version의 visibility 판단
        * visible → 현재 row 반환
        * not visible → roll_pointer를 따라 undo log 탐색하며 과거 버전을 찾음