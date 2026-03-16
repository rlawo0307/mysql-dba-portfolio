# 목차
* autocommit 동작 확인
* 트랜잭션 시작 시점 확인
* DDL implicit commit 동작 확인
* 참고 자료
    * [→ 트랜잭션에 대한 내용 자세히 보기](../transaction-basics.md)
    * [→ tcl에 대한 내용 자세히 보기](../../sql-commands/sql-tcl.md)
<br><br>

# autocommit 동작 확인
* autocommit이 활성화된 상태에서 각 SQL문이 자동으로 commit되는지 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
### session 1
```bash
mysql -u user1 -h localhost -p db1
```
```sql
select connection_id(); -- 8

-- autocommit 설정
show variables like 'autocommit'; -- default : autocommit on
set autocommit = 1; -- autocommit이 꺼져있다면 켜주기

create table t1(c1 int, c2 int);
insert into t1 values (1, 100);
```
### session 2
```bash
mysql -u user1 -h localhost -p db1
```
```sql
select connection_id(); -- 9

select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
+------+------+
1 row in set (0.00 sec)
```
## 결과
* session1에서 생성한 테이블과 입력한 데이터를 session2에서 확인 가능
## 결론
autocommit이 활성화되어 있는 경우, 각 SQL문은 하나의 트랜잭션처럼 동작하여 실행 후 자동으로 커밋된다. 따라서 명시적인 commit 없이도 다른 세션에서 변경 내용을 즉시 조회할 수 있다.
<br><br>

# 트랜잭션 시작 시점 확인
* autocommit이 비활성화 되어 있는 상태에서, 명시적으로 트랜잭션을 시작하지 않아도 자동으로 트랜잭션이 시작하는 것을 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
### session 1
```sql
-- 실험 준비
create table t1(c1 int, c2 int); -- DDL은 implicit commit을 발생시킴
                                 -- 실험 전에 미리 테이블 생성
-- 실험 시작
set autocommit = 0; -- autocommit 비활성화
insert into t1 values (1, 100);
```
### session 2
```sql
select * from t1;
```
```sql
Empty set (0.00 sec) -- 조회되는 row 없음
```
### session 1
```sql
insert into t1 values (2, 200);
commit; -- 여기서 commit 수행
```
### session 2
```sql
select * from t1;
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
## 결과
* session1에서 insert 실행 후 commit을 수행하기 전에는 session2에서 데이터가 조회되지 않음
* 이후 commit 수행 후 session2에서 데이터 조회 가능
## 결론
MySQL에서는 autocommit이 비활성화 된 상태에서 첫 DML이 실행되면 자동으로 트랜잭션을 시작한다. 따라서 명시적으로 트랜잭션을 시작하지 않아도 DML 실행(첫 insert) 이후에는 트랜잭션이 활성화된 상태가 된다.
<br><br>

# DDL implicit commit 동작 확인
* MySQL에서 자동으로 commit을 발생시키는 명령을 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
### session 1
```sql
-- 실험 준비
create table t1(c1 int, c2 int); -- DDL은 implicit commit을 발생시킴
                                 -- 실험 전에 미리 테이블 생성
-- 실험 시작
set autocommit = 0; -- autocommit 비활성화
insert into t1 values (1, 100); -- 첫 insert : 트랜잭션 활성화
```
### session 2
```sql
select * from t1;
```
```sql
Empty set (0.00 sec) -- 조회되는 row 없음
```
### session 1
```sql
create table t2(c1 int, c2 int); -- 새로운 table 생성(DDL)
```
### session 2
```sql
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+     -- 데이터 조회 가능
|    1 |  100 |
+------+------+
1 row in set (0.00 sec)
```
### session 1
```sql
insert into t1 values (2, 200); -- 두 번째 insert : 트랜잭션 활성화
create table t3(c1 int, c2 int); -- 새로운 table 생성(DDL)
rollback;
```
### session 2
```sql
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+     -- 두 번째 insert 데이터 조회 가능
|    1 |  100 |
|    2 |  200 |
+------+------+
2 rows in set (0.00 sec)
```
## 결과
* session1에서 첫 insert 실행 후
    * commit을 수행하지 않은 상태에서는 session2에서 데이터가 조회되지 않음
* session1에서 DDL(`create table t2`) 실행 후
    * commit을 수행하지 않아도 DDL 실행 이후 session2에서 데이터 조회 가능
* session1에서 두번째 insert와 DDL(`create table t3`) 실행 후
    * rollback을 수행했지만, DDL 실행 이전 변경 사항(두번째 insert)은 rollback 되지 않고 session2에서 그대로 조회됨
## 결론
DDL은 implicit commit을 발생시킨다. MySQL에서 DDL을 실행하면 내부적으로 commit → DDL → commit 순서로 수행된다.

session1에서 autocommit이 비활성화된 상태에서 첫 insert 수행하면 트랜잭션이 활성화된다. 이후 DDL(`create table t2`) 실행 시 implicit commit이 발생하므로, 명시적으로 commit을 수행하지 않아도 session1의 변경 사항을 session2에서 조회할 수 있다.

마찬가지로 session1의 두 번째 insert를 수행하면 새로운 트랜잭션이 자동으로 활성화된다. 이후 DDL(`create table t3`) 실행 시 implicit commit이 발생하며, 그 뒤 rollback을 수행해도 마지막 commit된 DDL 이전의 두 번째 insert는 이미 커밋되어 session2에서 그대로 조회된다.
<br>