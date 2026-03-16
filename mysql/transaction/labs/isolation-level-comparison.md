# 목차
* Isolation level 확인 및 변경
* Isolation level 별 동작 비교
* 참고 자료
    * [→ 트랜잭션에 대한 내용 자세히 보기](../transaction-basics.md)
    * [→ isolation level에 대한 내용 자세히 보기](../isolation-levels.md)
<br><br>

# Isolation level 확인 및 변경
* MySQL에서 격리 수준은 글로벌/세션 단위로 변경 가능
    * 트랜잭션 단위로 변경할 수 없음
* 글로벌 단위 변경 시
    * 이미 연결된 세션에는 영향 없음
    * 이후 새로 연결되는 모든 세션에 적용됨
    * 시스템 전체 동작에 영향을 주므로 운영 환경에서는 주의 필요
### 현재 isolation level 확인
```sql
select @@transaction_isolation;
```
```sql
+-------------------------+
| @@transaction_isolation |
+-------------------------+
| REPEATABLE-READ         | -- default
+-------------------------+
1 row in set (0.00 sec)
```
### isolation level 변경
```sql
-- 글로벌 단위 변경
set global transaction isolation level read uncommitted;
set global transaction isolation level read committed;
set global transaction isolation level repeatable read;
set global transaction isolation level serializable;

-- 세션 단위 변경
set session transaction isolation level read uncommitted;
set session transaction isolation level read committed;
set session transaction isolation level repeatable read;
set session transaction isolation level serializable;
```
<br><br>

# Isolation level 별 동작 비교
## Read Uncommitted
* commit, rollback 여부와 상관없이 다른 트랜잭션에서 데이터 조회 가능
* snapshot이 생성되지 않음
### session 1
```sql
set session transaction isolation level read uncommitted;

create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);

START TRANSACTION; -- 트랜잭션 활성화(snapshot 없음)
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
### session 2
```sql
SET autocommit = 0;

update t1 set c2 = 2000 where c1 = 2;
-- 아직 commit 안함
```
### session 1
```sql
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 | 2000 | -- 변경된 데이터가 조회됨
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
## Read Committed
* commit이 완료된 데이터만 다른 트랜잭션에서 조회 가능
* select 쿼리마다 새로운 snapshot 생성(InnoDB 기준)
### session 1
```sql
set session transaction isolation level read committed;

create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);
START TRANSACTION; -- 트랜잭션 활성화
```
### session 2
```sql
SET autocommit = 0;

update t1 set c2 = 2000 where c1 = 2;
-- 아직 commit 안함
```
### session 1
```sql
select * from t1; -- select 쿼리마다 새로운 snapshot 생성
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 | -- 변경되지 않은 초기 데이터가 조회됨
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
### session 2
```sql
commit;
```
### session 1
```sql
select * from t1; -- select 쿼리마다 새로운 snapshot 생성
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 | 2000 | -- 변경된 데이터 조회됨
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
## Repeatable Read
* 같은 트랜잭션 내에서 동일한 select에 대해 항상 같은 결과를 보장
* 트랜잭션 시작 시 snapshot 생성
* 트랜잭션이 끝날 때까지 처음 생성된 snapshot 유지
### session 1
```sql
set session transaction isolation level repeatable read;

create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);
START TRANSACTION; -- 트랜잭션 활성화(snapshot 생성)
```
### session 2
```sql
SET autocommit = 0;

update t1 set c2 = 2000 where c1 = 2;
-- 아직 commit 안함
```
### session 1
```sql
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 | -- 변경되지 않은 초기 데이터가 조회됨
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
### session 2
```sql
commit;
```
### session 1
```sql
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 | -- 변경되지 않은 초기 데이터가 조회됨
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
## Serializable
* 모든 트랜잭션을 직렬 실행한 것과 동일한 결과를 보장
* 트랜잭션 시작 시 snapshot 생성
* 모든 select가 lock 획득
* 동작
    * lock 충돌이 없을 경우
        * 생성한 snapshot을 기준으로 데이터 조회
    * lock 충돌이 발생할 경우
        * lock이 해제된 시점의 최신 커밋 데이터를 조회
        * 새로운 snapshot이 생성되는 것은 아님 
### session 1
```sql
set session transaction isolation level serializable;

create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);
START TRANSACTION; -- 트랜잭션 활성화(snapshot 생성)
```
### session 2
```sql
SET autocommit = 0;

update t1 set c2 = 2000 where c1 = 2;
-- 아직 commit 안함
```
### session 1
```sql
select * from t1;
-- 모든 select가 lock을 획득해야 함
-- lock 충돌 발생
-- session2가 commit/rollback해야 진행 가능
-- lock 대기 상태
```
### session 2
```sql
commit; -- session2가 lock 반납
```
### session 1
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 | 2000 | -- 변경된 데이터 조회됨
|    3 |  300 | -- session2의 커밋이 반영된 것처럼 동작       
+------+------+  
3 rows in set (11.49 sec)
```
## 결론
<center>

|                    |commit 전|commit 후|데이터 변경 여부 (O/X)|
|--------------------|:--------:|:-----------------:|:-----:|
|**READ UNCOMMITTED**|변경된 데이터 조회|변경된 데이터 조회|O / O|
|**READ COMMITTED**  |초기 데이터 조회|변경된 데이터 조회|X / O|
|**REPEATABLE READ** |초기 데이터 조회|초기 데이터 조회|X / X|
|**SERIALIZABLE**    |lock 대기|변경된 데이터 조회|- / O|
</center>
<br>