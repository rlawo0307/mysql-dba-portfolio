# 목차
* Lock Wait 상황 분석
    * 실습 환경 준비
    * 정상 상태 확인
    * 문제 상황 발생
    * 상태 분석
        * 이상 감지
        * 트랜잭션 확인
        * blocking 관계 확인
        * lock 대상 및 종류 확인
        * lock 구조 및 상세 분석
        * 전체 결과를 빠르게 확인
* 참고 자료
    * [모니터링이란?](../monitoring.md)
<br><br>

# Lock Wait 상황 분석
### 0. 실습 환경 준비
```sql
create table t1(c1 int primary key, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);
```
### 1. 정상 상태 확인
* 전체적으로 replication 관련 내부 thread들이 존재
* 현재 처리할 이벤트가 없어 유휴 상태로 대기 중인 정상 상태로 해석 가능
```sql
show processlist;
```
```sql
+----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+------------------+
| Id | User            | Host            | db   | Command | Time | State                                                    | Info             |
+----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+------------------+
|  5 | system user     | connecting host | NULL | Connect | 1445 | Connecting to source                                     | NULL             |
|  6 | event_scheduler | localhost       | NULL | Daemon  | 1445 | Waiting on empty queue                                   | NULL             |
|  7 | system user     |                 | NULL | Query   | 1445 | Replica has read all relay log; waiting for more updates | NULL             |
| 11 | system user     |                 | NULL | Connect | 1445 | Waiting for an event from Coordinator                    | NULL             |
| 12 | system user     |                 | NULL | Connect | 1445 | Waiting for an event from Coordinator                    | NULL             |
| 13 | system user     |                 | NULL | Connect | 1445 | Waiting for an event from Coordinator                    | NULL             |
| 14 | system user     |                 | NULL | Connect | 1445 | Waiting for an event from Coordinator                    | NULL             |
| 44 | user1           | localhost       | db1  | Query   |    0 | init                                                     | show processlist |
+----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+------------------+
8 rows in set, 1 warning (0.00 sec)
```
* `id = 5` : MySQL 내부 thread(replication I/O thread)가 source 서버와 연결을 유지하며 통신 대기 중
* `id = 6` : event_scheduler thread가 background에서 실행할 이벤트가 없어 대기 중
* `id = 7` : MySQL 내부 thread(replication SQL thread)가 relay log를 모두 적용한 후 새로운 이벤트를 기다리는 중
* `id = 11 ~ 14` :  MySQL 내부 thread(병렬 복제 worker thread)들이 coordinator로부터 작업을 기다리는 중
* `id = 44` : user1이 db1에서 show processlist 쿼리 실행 중
### 2. 문제 상황 발생
* 동일한 row(`c1=1`)에 대해 두 트랜잭션이 update를 수행하면서 lock 충돌 발생
#### session 1
```sql
set autocommit = 0;

start transaction;
update t1 set c2 = 999 where c1 = 1;
-- 여기서 commit 안함
```
#### session 2
```sql
update t1 set c2 = 500 where c1 = 1; -- blocked
```
### 3. 상태 분석
```text 
[모니터링 루틴]

1. processlist                          → 이상 감지
2. innodb_trx                           → 트랜잭션 확인
3. data_lock_waits                      → blocking 관계 확인
4. data_locks                           → lock 대상 및 종류 확인
5. show engine innodb status            → lock 구조 및 상세 분석
6. sys schema                           → 전체 결과를 빠르게 확인
```
#### 1) 이상 감지
```sql
show processlist;
```
```sql
+-----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+-------------------------------------+
| Id  | User            | Host            | db   | Command | Time | State                                                    | Info                                |
+-----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+-------------------------------------+
| 110 | user2           | localhost       | db1  | Query   |    2 | updating                                                 | update t1 set c2 = 500 where c1 = 1 |
+-----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+-------------------------------------+
9 rows in set, 1 warning (0.00 sec)
```
* `id = 110` : user2가 update 쿼리를 2초 동안 수행중. 해당 row의 lock이 해제되기를 기다리는 상태일 가능성이 높음
    * 단, lock wait이 아닌 실제 update에 시간이 많이 걸리는 거일 수도 있음
    * 실제로 blocking인지 확정하기 위해서는 추가적인 확인이 필요
#### 2) 트랜잭션 확인
```sql
select * from information_schema.innodb_trx\G
```
```sql
-- 해석에 필요한 부분만 발췌
*************************** 1. row ***************************
                    trx_id: 71292
                 trx_state: LOCK WAIT
          trx_wait_started: 2026-04-21 06:38:45
       trx_mysql_thread_id: 110
                 trx_query: update t1 set c2 = 500 where c1 = 1
         trx_rows_modified: 0
*************************** 2. row ***************************
                    trx_id: 71269
                 trx_state: RUNNING
          trx_wait_started: NULL
       trx_mysql_thread_id: 44
                 trx_query: select * from information_schema.innodb_trx
         trx_rows_modified: 1
2 rows in set (0.00 sec)
```
* 110번 트랜잭션은 update 쿼리 수행을 위해 lock을 기다리는 상태
* 44번 트랜잭션은 select 조회를 수행 중
    * **44번 트랜잭션이 실제로 lock을 보유한 blocking 트랜잭션인지 알 수 없음**
    * lock을 보유한 트랜잭션을 확정하기 위해서는 추가적인 확인이 필요
#### 3) blocking 관계 확인
```sql
select * from performance_schema.data_lock_waits;
```
```sql
-- 해석에 필요한 부분만 발췌
REQUESTING_ENGINE_TRANSACTION_ID  : 71292 -- lock을 기다리는 트랜잭션
BLOCKING_ENGINE_TRANSACTION_ID    : 71269 -- lock을 잡고있는 트랜잭션
```
#### 4) lock 대상 및 종류 확인
```sql
select ENGINE_TRANSACTION_ID as TX, concat(OBJECT_SCHEMA, '.', OBJECT_NAME) as OBJ, INDEX_NAME, LOCK_TYPE, LOCK_MODE, LOCK_STATUS, LOCK_DATA from performance_schema.data_locks;
```
```sql
+-------+--------+------------+-----------+---------------+-------------+-----------+
| TX    | OBJ    | INDEX_NAME | LOCK_TYPE | LOCK_MODE     | LOCK_STATUS | LOCK_DATA |
+-------+--------+------------+-----------+---------------+-------------+-----------+
| 71269 | db1.t1 | NULL       | TABLE     | IX            | GRANTED     | NULL      | -- IX = Intention X lock
| 71292 | db1.t1 | NULL       | TABLE     | IX            | GRANTED     | NULL      | -- IX = Intention X lock
| 71269 | db1.t1 | PRIMARY    | RECORD    | X,REC_NOT_GAP | GRANTED     | 1         | -- pk = 1 인 row에 대한 X lock을 보유하고 있음
| 71292 | db1.t1 | PRIMARY    | RECORD    | X,REC_NOT_GAP | WAITING     | 1         | -- 같은 row에 대해 lock을 요청하다가 대기 중
+-------+--------+------------+-----------+---------------+-------------+-----------+
```
#### 5) lock 구조 및 상세 분석
```sql
show engine innodb status\G
```
```sql
-- 해석에 필요한 부분만 발췌
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 71292, ACTIVE 2 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1128, 1 row lock(s)
MySQL thread id 110, OS thread handle 136427288442560, query id 108 localhost user2 updating
update t1 set c2 = 500 where c1 = 1
------- TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 464 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 71292 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 000000011665; asc      e;;
 2: len 7; hex 02000001a42bd7; asc      + ;;
 3: len 4; hex 800003e7; asc     ;;
```
* `TRANSACTION 71292, ACTIVE 2 sec starting index read`
    * 71292 트랜잭션이 2초째 활성화 상태
    * 인덱스를 따라 row를 찾는 중
* `mysql tables in use 1, locked 1`
    * 테이블 1개를 사용 중
    * lock 구조를 가지고 있음
* `LOCK WAIT 2 lock struct(s), heap size 1128, 1 row lock(s)`
    * 현재 lock 대기 중
    * row 단위 lock과 관련된 대기 상황
* `MySQL thread id 110, ... user2 updating`
    * 현재 대기 중인 트랜잭션의 thread_id는 110
    * 사용자는 user2
    * `update t1 set c2 = 500 where c1 = 1` 쿼리에서 문제 발생
* `TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED:`
    * 이 트랜잭션은 2초째 lock을 기다리는 중
* `RECORD LOCKS ... index PRIMARY of table `db1`.`t1` trx id 71292`
    * 71292 트랜잭션은 db1.t1 테이블의 primary 인덱스에서 record lock을 기다리는 중
    * 즉, pk 기준으로 특정 row를 찾은 뒤 그 row lock을 기다리고 있는 상황
* `lock_mode X locks rec but not gap waiting`
    * update이므로 X lock 필요
    * gap을 잠그는 next-key lock이나 gap lock이 아닌 record lock을 걸 예정
    * 즉, 단일 row에 대한 X record lock 대기 중
* `Record lock, heap no 2 PHYSICAL RECORD: n_fields 4`
    * 실제로 lock이 걸려있는 물리 레코드 정보
    * 이 row는 컬럼 4개로 구성
        * ` 0: len 4; hex 80000001`
            * hex 80000001 = 1
            * 즉, pk = 1
        * `1: len 6; hex 000000011665`
            * 보통 transaction id 또는 row version 정보
            * InnoDB 내부 MVCC용
            * 사용자 컬럼 아님
        * `2: len 7; hex 02000001a42bd7`
            * roll pointer(undo log 위치)
            * 내부 관리용 컬럼
        * `3: len 4; hex 800003e7`
            * hex 800003e7 = 999
            * 즉, c2 = 999
    * 해석하면 (1, 999)가 lock 대상 row임을 알 수 있음
#### 6) 전체 결과를 빠르게 확인
```sql
select * from sys.innodb_lock_waits;
```
```sql
-- 해석에 필요한 부분만 발췌
wait_age_secs       : 2             -- lock 대기 시간(초)
locked_table        : db1.t1        -- lock이 걸린 테이블
locked_index        : PRIMARY       -- lock이 걸린 인덱스
locked_type         : RECORD        -- lock 걸리는 대상
waiting_trx_id      : 71292         -- lock을 기다리는 트랜잭션 id
waiting_pid         : 110           -- lock을 기다리는 세션 id
waiting_query       : update t1 set c2 = 500 where c1 = 1 -- lock을 기다리는 쿼리
waiting_lock_mode   : X,REC_NOT_GAP -- lock을 기다리는 트랜잭션이 요청한 lock 종류
blocking_trx_id     : 71269         -- lock을 보유하고 있는 트랜잭션 id
blocking_pid        : 44            -- lock을 보유하고 있는 세션 id
blocking_lock_mode  : X,REC_NOT_GAP  -- lock을 보유하고 있는 트랜잭션이 건 lock 종류
```