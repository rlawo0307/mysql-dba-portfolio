# 목차
* MDL 대기 상황 분석 실습
    * MDL 대기 상황 재현
    * MDL 상태 확인
    * MDL 대기 세션 확인
    * MDL 대기 해결
* 참고 자료
    * [MDL 이란](../MDL.md)
<br><br>

# MDL 대기 상황 분석 실습
* DDL이 MDL 때문에 대기하는 상황을 재현해보자
* `performance_schema.metadata_locks`를 통해 보유 중인 lock과 대기 중인 lock을 분석해보자
* `show processlist`와 `performance_schema.threads`를 이용하여 MDL 보유 세션을 식별해보자
* MDL 대기 상태를 해결해보자
## MDL 대기 상황 재현
### session 1
```sql
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);

start transaction;
select * from t1 where c1 = 1; -- shared MDL 획득 후 트랜잭션 유지 (commit 안함)
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
+------+------+
1 row in set (0.00 sec)
```
### session 2
```sql
alter table t1 add column c3 int; -- exclusive MDL이 필요하므로 대기
```
## MDL 상태 확인
* 시스템 테이블은 root 계정 또는 조회 권한이 있는 계정에서 조회 가능
```sql
select object_type, object_schema, object_name, lock_type, lock_duration, lock_status
from performance_schema.metadata_locks
where object_name = 't1';
```
```sql
+-------------+---------------+-------------+-------------------+---------------+-------------+
| object_type | object_schema | object_name | lock_type         | lock_duration | lock_status |
+-------------+---------------+-------------+-------------------+---------------+-------------+
| TABLE       | db1           | t1          | SHARED_READ       | TRANSACTION   | GRANTED     |
| TABLE       | db1           | t1          | SHARED_UPGRADABLE | TRANSACTION   | GRANTED     |
| TABLE       | db1           | t1          | EXCLUSIVE         | TRANSACTION   | PENDING     |
+-------------+---------------+-------------+-------------------+---------------+-------------+
3 rows in set (0.00 sec)
```
* `SHARED_READ`
    * session 1이 select 수행 시 획득한 shared MDL
* `SHARED_UPGRADABLE`
    * session 2가 DDL 수행 과정에서 획득한 MDL
    * session 1이 보유한 shared MDL과 호환되므로 획득 가능
    * 이후 exclusive MDL 획득을 시도할 수 있음
* `EXCLUSIVE`
    * session 2가 최종적으로 필요한 MDL
    * session 1이 보유한 shared MDL과 호환되지 않으 
    * 따라서 `PENDING` 상태로 대기 중
## MDL 대기 세션 확인
### processlist 조회하기
```sql
show processlist;
```
```sql
+----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+----------------------------------+
| Id | User            | Host            | db   | Command | Time | State                                                    | Info                             |
+----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+----------------------------------+
|  5 | system user     | connecting host | NULL | Connect |  510 | Connecting to source                                     | NULL                             |
|  6 | system user     |                 | NULL | Query   |  510 | Replica has read all relay log; waiting for more updates | NULL                             |
|  7 | event_scheduler | localhost       | NULL | Daemon  |  510 | Waiting on empty queue                                   | NULL                             |
| 10 | system user     |                 | NULL | Connect |  510 | Waiting for an event from Coordinator                    | NULL                             |
| 11 | system user     |                 | NULL | Connect |  510 | Waiting for an event from Coordinator                    | NULL                             |
| 12 | system user     |                 | NULL | Connect |  510 | Waiting for an event from Coordinator                    | NULL                             |
| 13 | system user     |                 | NULL | Connect |  510 | Waiting for an event from Coordinator                    | NULL                             |
| 14 | user1           | localhost       | db1  | Sleep   |  239 |                                                          | NULL                             | -- 14번 세션이 MDL 보유 세션으로 의심됨
| 15 | user1           | localhost       | db1  | Query   |  228 | Waiting for table metadata lock                          | alter table t1 add column c3 int | -- 15번 세션이 MDL을 기다리는 중
| 17 | root            | localhost       | NULL | Query   |    0 | init                                                     | show processlist                 |
+----+-----------------+-----------------+------+---------+------+----------------------------------------------------------+----------------------------------+
10 rows in set, 1 warning (0.00 sec)
```
* processlist에서 `Sleep` 상태라고 해서 무조건 MDL 보유 세션이라고 단정할 수 없음
* 정확한 판단을 위해서는 performance_schema의 `metadata_locks`와 `threads`를 join해서 확인해 봐야 함
    * `owner_thread_id` : performance_schema 내부 thread id
    * `processlist_id` : show processlist에서 확인 가능한 세션 id 
### performance_schema 조회하기
```sql
select object_type, object_schema, object_name, lock_type, lock_duration, lock_status, owner_thread_id, processlist_id
from performance_schema.metadata_locks as ml
join performance_schema.threads th
on ml.owner_thread_id = th.thread_id
where object_name = 't1';
```
```sql
+-------------+---------------+-------------+-------------------+---------------+-------------+-----------------+----------------+
| object_type | object_schema | object_name | lock_type         | lock_duration | lock_status | owner_thread_id | processlist_id |
+-------------+---------------+-------------+-------------------+---------------+-------------+-----------------+----------------+
| TABLE       | db1           | t1          | SHARED_READ       | TRANSACTION   | GRANTED     |              53 |             14 | -- 14번 세션이 MDL이 보유중인것이 확실해짐
| TABLE       | db1           | t1          | SHARED_UPGRADABLE | TRANSACTION   | GRANTED     |              54 |             15 |
| TABLE       | db1           | t1          | EXCLUSIVE         | TRANSACTION   | PENDING     |              54 |             15 | -- 15번 세션이 MDL을 기다리는 중
+-------------+---------------+-------------+-------------------+---------------+-------------+-----------------+----------------+
3 rows in set (0.01 sec)
```
## MDL 대기 해결
### session 1
* MDL을 보유한 세션이 트랜잭션을 종료하면 lock이 해제됨
```sql
commit;
```
```sql
rollback;
```
### root
* 세션에 접근할 수 없는 경우 관리자(또는 권한이 있는 사용자)가 세션 종료 가능
```sql
kill 14; -- kill 명령은 processlist_id를 사용
```