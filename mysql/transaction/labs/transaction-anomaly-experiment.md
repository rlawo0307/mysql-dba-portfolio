# 목차
* Transaction Anomaly 재현 및 해결
    * Dirty Read
    * Non-repeatable Read
    * Phantom Read
* 참고 자료
    * [이상 현상 및 isolation level](../isolation-levels.md)
<br><br>

# Transaction Anomaly 재현 및 해결
* transaction isolation level에 따라 발생할 수 있는 이상 현상을 직접 재현해보자
* 각 이상 현상이 어떤 상황에서 발생하는지 확인해보자
* isolation level 변경 또는 locking read를 통해 이상 현상을 방지할 수 있는지 확인해보자
## Dirty Read - 재현
* commit되지 않은 데이터를 읽는 현상
* read uncommitted 격리 수준에서 발생 가능
### 실습 준비
```sql
rollback;
set autocommit = 1;

drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);
```
### session 1
```sql
set session transaction isolation level read uncommitted;
set autocommit = 0;

start transaction;
update t1 set c2 = 999 where c1 = 1;
-- 아직 commit하지 않음
```
### session 2
```sql
set session transaction isolation level read uncommitted;

start transaction;
select * from t1 where c1 = 1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  999 | -- 아직 commit되지 않은 데이터(c2 = 999)를 읽음
+------+------+
1 row in set (0.00 sec)
```
### session 1
```sql
rollback; -- 변경 내용 취소
```
### session 2
```sql
select * from t1 where c1 = 1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 | -- session 1 rollback 후 복구된 값(c2 = 100)을 읽음
+------+------+
1 row in set (0.00 sec)
```
## Dirty Read - 해결
* read committed 이상에서 dirty read를 방지할 수 있는지 확인해보자
### 실습 준비
```sql
rollback;
set autocommit = 1;

drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);
```
### session 1
```sql
set session transaction isolation level read committed;
set autocommit = 0;

start transaction;
update t1 set c2 = 999 where c1 = 1;
select * from t1 where c1 = 1;
-- 아직 commit하지 않음
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  999 |
+------+------+
1 row in set (0.00 sec)
```
### session 2
```sql
set session transaction isolation level read committed;

start transaction;
select * from t1 where c1 = 1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 | -- session 1의 commit 전 변경 값이 아닌 마지막 commit된 데이터(c2 = 100)를 읽음 (dirty read 발생하지 않음)
+------+------+
1 row in set (0.00 sec)
```
## Non-repeatable Read - 재현
* 하나의 트랜잭션 내부에서 같은 row를 두 번 조회했을 때 결과가 달라지는 현상
* read committed 격리 수준에서 발생 가능
### 실습 준비
```sql
rollback;
set autocommit = 1;

drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);
```
### session 1
```sql
set session transaction isolation level read committed;

start transaction; -- 트랜잭션 시작
select * from t1 where c1 = 1;
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
set session transaction isolation level read committed;
set autocommit = 0;

start transaction;
update t1 set c2 = 999 where c1 = 1;
commit;
```
### session 1
```sql
-- 아직 첫 번째 트랜잭션
select * from t1 where c1 = 1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  999 | -- 같은 트랜잭션에서 같은 row를 조회했지만 c2 값이 달라짐
+------+------+
1 row in set (0.00 sec)
```
## Non-repeatable Read - 해결
* repeatable read 이상에서 non-repeatable read를 방지할 수 있는지 확인해보자
* 단, for update 등과 같은 [current read](../consistent-vs-current-read.md)에서는 snapshot이 아닌 최신 committed row를 읽으므로 결과 값이 달라질 수 있음
### 실습 준비
```sql
rollback;
set autocommit = 1;

drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);
```
### session 1
```sql
set session transaction isolation level repeatable read;

start transaction; -- 트랜잭션 시작
select * from t1 where c1 = 1;
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
set session transaction isolation level repeatable read;
set autocommit = 0;

start transaction;
update t1 set c2 = 999 where c1 = 1;
commit;
```
### session 1
```sql
-- 아직 첫 번째 트랜잭션
select * from t1 where c1 = 1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 | -- session 2에서 commit했지만 session 1의 snapshot에서는 기존 값이 유지됨
+------+------+
1 row in set (0.01 sec)
```
## Phantom Read - 재현
* 하나의 트랜잭션 내부에서 같은 조건으로 두 번 조회했을 때 없던 row가 나타나는 현상
* SQL 표준에서는 repeatable read 격리 수준에서도 발생 가능
* InnoDB에서는 phantom read를 방지하는 메커니즘이 존재
    * 일반 select의 경우, MVCC 기반 consistent read로 방지
    * current read의 경우, next-key lock으로 phantom insert 방지
* read committed에서 phantom read를 재현해보고 repeatable read에서 방지되는지 확인해보자
### 실습 준비
```sql
rollback;
set autocommit = 1;

drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);
```
### session 1
```sql
set session transaction isolation level read committed;

start transaction; -- 트랜잭션 시작
select * from t1 where c2 >= 100;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
+------+------+
2 rows in set (0.00 sec)
```
### session 2
```sql
set session transaction isolation level read committed;
set autocommit = 0;

start transaction;
insert into t1 values (3, 300); -- session 1의 조회 조건에 해당하는 새로운 row를 insert
commit;
```
### session 1
```sql
-- 아직 첫 번째 트랜잭션
select * from t1 where c2 >= 100; -- 같은 조건으로 다시 조회
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 | -- 이전에 보이지 않았던 새로운 row가 조회됨
+------+------+
3 rows in set (0.00 sec)
```
## Phantom Read - 해결
* InnoDB에서 repeatable read에서 phantom read를 방지할 수 있는지 확인해보자
### 실습 준비
```sql
rollback;
set autocommit = 1;

drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);
```
### session 1
```sql
set session transaction isolation level repeatable read;

start transaction; -- 트랜잭션 시작
select * from t1 where c2 >= 100;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
+------+------+
2 rows in set (0.00 sec)
```
### session 2
```sql
set session transaction isolation level repeatable read;
set autocommit = 0;

start transaction;
insert into t1 values (3, 300); -- session 1의 조회 조건에 해당하는 새로운 row를 insert
commit;
```
### session 1
```sql
-- 아직 첫 번째 트랜잭션
select * from t1 where c2 >= 100; -- 같은 조건으로 다시 조회
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 | -- session 2에서 추가한 row는 session 1의 현재 snapshot에서는 보이지 않음
+------+------+
2 rows in set (0.00 sec)
```