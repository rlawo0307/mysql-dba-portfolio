# Binlog란?
* MySQL에서 발생한 데이터 변경 이력(DDL, DML)을 기록하는 로그
* 단순히 data file 자체를 복제하는 것이 아니라, 변경 내용을 이벤트 단위로 기록
* replication, point-in-time recovery(PITR)의 핵심 기반
## Binlog 구조
```text
binlog file = 로그 파일 전체
event = 로그 한 줄
transaction = 관련 로그 묶음
```
* 하나의 binlog file은 여러 개의 binlog event로 구성
* 하나의 트랜잭션은 여러 개의 binlog event의 집합으로 표현
* 단, 모든 binlog event가 트랜잭션에 속하는 것은 아님
### 전체 구조
* binlog는 event가 순차적으로 나열된 flat 구조
* 트랜잭션은 이 event들의 일부를 묶어서 해석한 논리적 단위
```text
binlog file
 ├── Format Description Event
 ├── Previous GTIDs Event
 ├── event  (transaction 1 시작)
 ├── event
 ├── event
 ├── Xid event (commit)
 ├── event  (transaction 2 시작)
 ├── event
 ├── Xid event (commit)
 └── ...
```
* `Format Description Event`
    * binlog 파일의 구조와 포맷을 정의하는 metadata event
    * 파일 맨 처음에 등장
    * binlog를 해석하기 위한 기준 정보 제공
        * binlog 버전
        * event 포맷
        * 각 event 타입 정보 등
* `Previous GTIDs Event`
    * 해당 binlog file이 시작되기 이전까지 실행된 GTID 집합을 담고 있는 metadata event
    * 파일 초반에 등장
    * replication에서 어떤 트랜잭션을 이미 적용했는지 판단하는 기준
        * 내(replica)가 가진 GTID 확인
        * source binlog를 통해 source에서 실행된 GTID 확인
        * 없는 GTID만 실행
* `Xid event (commit)`
    * 트랜잭션의 끝을 구분하는 event
    * 트랜잭션 commit을 의미하는 event
## Binlog 기록 방식
### SBR (Statement-Based Replication)
```text
Query_event:
UPDATE t1 SET c2 = c2 + 1 WHERE c1 = 10;
```
* source에서는 SQL문 자체를 기록
* replica는 기록된 SQL을 실행
* 단순하고 읽기 쉬움
* 로그 크기가 작음
* 실행 환경이나 데이터 상태에 따라 결과가 달라질 수 있음
* 비결정적 함수 사용 시 replica에서 다른 결과가 나올 수 있음
### RBR (Row-Based Replication)
```text
Table_map_event:
  table_id = 123 → t1

Update_rows_event:
  before: (c1=10, c2=100)
  after : (c1=10, c2=101)

Xid_event (commit)
```
* source에서는 변경 된 row 데이터를 기록
* replica는 변경 내용을 데이터에 적용
* MySQL 8 버전에서는 RBR이 default
* replication 안정성이 높음
* 로그 크기가 큼
* ex) `before : id = 10, c1 = 100 / after  : id = 10, c1 = 101`
### Mixed
* 기본적으로 SBR을 적용
* 비결정적 상황에서는 RBR로 자동 전환
<br><br>

# GTID란?
```text
GTID = server_uuid:transaction_id
```
* 트랜잭션에 부여되는 전역 고유 ID
* 기존의 binlog file + position 기반 방식의 문제를 해결하기 위해 등장
    * failover 시 위치를 찾기가 어려움
    * 사람이 직접 맞춰야 함
* 장점
    * failover 자동화 가능
    * replication 재구성 쉬움
    * 데이터 일관성 보장
* 주의할 점
    * GTID 모드를 켜면 되돌리기 어려움
    * 모든 트랜잭션이 GTID를 가져야 함
    * 일부 DDL/관리 작업에서는 제약이 있음