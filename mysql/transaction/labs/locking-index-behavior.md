# 목차
* Index 유무에 따른 lock 범위 비교
    * 다른 row에 접근할 경우
    * 같은 row에 접근할 경우
    * 인덱스가 존재하지만 사용하지 않을 경우
* Index 종류 별 lock 범위 확인
    * unique index → `record lock`
    * non-unique index → `next-key lock`
* Index 종류 별 lock 전파 확인
    * secondary index
    * covering index
* 참고 자료
    * [→ 트랜잭션에 대한 내용 자세히 보기](../transaction-basics.md)
    * [→ lock에 대한 내용 자세히 보기](../locking.md)
<br><br>

# Index 유무에 따른 lock 범위 비교(1)
* 인덱스 유무에 따라 InnoDB의 lock 범위가 어떻게 달라지는지 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
* session2는 session1과 다른 row에 접근
    * 동일 row 충돌을 배제
* 인덱스는 pk(unique index)를 사용
## index가 없을 때(table scan)
### session 1
```sql
-- 실험 준비
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화

start transaction; -- 트랜잭션 시작
select * from t1 where c1 = 2 for update; -- session1은 c1=2 인 row에 접근
-- 여기서 커밋 안함
```
### session 2
```sql
update t1 set c2 = 999 where c1 = 3; -- session2는 c1=3 인 row에 접근
```
```sql
-- blocked되어 update 수행 지연
```
## index가 있을 때
### session 1
```sql
-- 실험 준비
create table t1(c1 int primary key, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화

start transaction; -- 트랜잭션 시작
select * from t1 where c1 = 2 for update; -- session1은 c1=2 인 row에 접근
```
### session 2
```sql
update t1 set c2 = 999 where c1 = 3; -- session2는 c1=3 인 row에 접근
```
```sql
Query OK, 1 row affected (0.00 sec) -- blocked되지 않고 update 수행 완료
Rows matched: 1  Changed: 1  Warnings: 0
```
## 결과
* index가 없을 때 : session1에서 커밋하기 전에는 session2는 blocked됨
* index가 있을 때 : session1에서 커밋하기 전이지만 session2는 blocked되지 않고 update 수행
## 결론
실험 결과, 인덱스 유무에 따라 session2의 update문의 blocked 여부가 결정되는 것을 확인할 수 있다. 이는 인덱스 유무에 따라 lock이 걸리는 범위가 결정되기 때문이다.

인덱스가 없는 경우, InnoDB는 full table scan을 수행하며 모든 row를 검사하는 과정에서 lock을 획득한다. 그 결과, 조건과 무관한 다른 row에 대한 작업도 blocking되게 된다.

인덱스가 있는 경우, 인덱스를 통해 필요한 row에만 접근하여 lock을 획득한다. 특히 primary key와 같은 unique index를 사용할 경우, 특정 row에 대한 record lock만 발생하며 다른 row에 대한 작업은 blocking되지 않는다.

결과적으로, lock 범위는 단순한 쿼리 조건이 아니라 실행 계획과 인덱스 사용 여부에 의해 결정된다. 

현재 실험은 session1과 session2가 서로 다른 row에 접근하여 존재하는 인덱스를 그대로 사용하였다. 따라서, 인덱스가 존재하지만 사용되지 않은 경우나 같은 row에 접근하는 상황에 대한 추가적인 실험이 필요하다.
<br><br>

# Index 유무에 따른 lock 범위 비교(2)
* session1과 session2가 같은 row에 접근할 경우, lock 범위가 어떻게 달라지는지 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
* 인덱스는 pk(unique index)를 사용
## index가 없을 때(table scan)
* Index 유무에 따른 lock 범위 비교(1)에서 full table scan을 수행하여 session2가 blocked되는 것을 확인하였으므로 생략
## index가 있을 때
### session 1
```sql
-- 실험 준비
create table t1(c1 int primary key, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화

start transaction; -- 트랜잭션 시작
select * from t1 where c1 = 2 for update;
```
### session 2
```sql
update t1 set c2 = 999 where c1 = 2; -- session1과 같은 row에 접근
```
```sql
-- blocked되어 update 수행 지연
```
## 결과
* session1에서 커밋하기 전에는 session2는 blocked됨
## 결론
실험 결과, 서로 다른 세션이 같은 row에 접근할 경우 인덱스가 존재하더라도 무조건 lock 충돌이 발생하는 것을 확인할 수 있다.

InnoDB는 row 단위로 exclusive lock(X lock)을 관리한다. 따라서 동일한 row에 대해 동시에 수정 작업을 수행하는 것은 허용되지 않는다. 

즉, 인덱스는 lock 범위를 줄이는 역할을 할 뿐, 동일 row에 대한 동시 접근 충돌을 방지하지 않는다.
<br><br>

# Index 유무에 따른 lock 범위 비교(3)
* 인덱스가 존재하지만 사용되지 않을 경우, lock 범위가 어떻게 달라지는지 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
* session2는 session1과 다른 row에 접근
* 인덱스는 secondary index + non-unique index 사용
## index가 없을 때(table scan)
* Index 유무에 따른 lock 범위 비교(1)에서 full table scan을 수행하여 session2가 blocked되는 것을 확인하였으므로 생략
## index가 있을 때
### session 1
```sql
-- 실험 준비
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int, index idx1(c1));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select 2, 200 from seq; -- skew한 데이터 삽입
insert into t1 values (3, 300);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화

start transaction; -- 트랜잭션 시작
select * from t1 where c1 >= 2 and c1 < 3 for update;
explain select * from t1 where c1 >= 2 and c1 < 3 for update;
```
```sql
type            : ALL -- full table scan
possible_keys   : idx1 -- 사용 가능한 인덱스는 존재 
key             : NULL -- 실제로는 인덱스 미사용
key_len         : NULL
Extra           : NULL
```
### session 2
```sql
update t1 set c2 = 999 where c1 = 3;
```
```sql
-- blocked되어 update 수행 지연
```
## 결과
* session1에서 커밋하기 전에는 session2는 blocked됨
## 결론
실험 결과, 인덱스가 존재하고 인덱스가 걸려 있는 컬럼을 조건으로 사용하더라도 옵티마이저가 인덱스를 사용하는 것보다 full table scan의 cost가 더 낮다고 판단하는 경우, 테이블을 scan하는 과정에서 모든 row에 대해 순차적으로 lock을 획득한다. 그 결과 조건과 무관한 다른 row에 대한 작업도 blocking되는 현상이 발생한다.

결론적으로, InnoDB에서 lock 범위는 단순히 인덱스 존재 여부가 아닌 실제 실행 계획에서 선택된 접근 방식에 의해 결정된다.
<br><br>

# Index 종류 별 lock 범위 비교
* 인덱스 종류에 따라 InnoDB의 lock 범위가 어떻게 달라지는지 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
* session2는 session1과 다른 row에 접근
    * 동일 row 충돌을 배제
## unique index
### session 1
```sql
-- 실험 준비
create table t1(c1 int, c2 int, unique index idx1(c1));
insert into t1 values (10, 100), (20, 200), (30, 300);

-- 실험 시작
set autocommit = 0;

start transaction;
select * from t1 where c1 = 20 for update;
```
### session 2
* record lock 확인
```sql
update t1 set c2 = 999 where c1 = 20;
```
```sql
-- blocked되어 update 수행 지연, record lock 걸림
```
* gap lock 확인
```sql
insert into t1 values (15, 150);
```
```sql
Query OK, 1 row affected (0.00 sec) -- insert 성공, gap lock 없음
```
## non-unique index
### session 1
```sql
-- 실험 준비
create table t1(c1 int, c2 int, index idx1(c1));
insert into t1 values (10, 100), (20, 200), (30, 300);

-- 실험 시작
set autocommit = 0;

start transaction;
select * from t1 where c1 = 20 for update;
```
### session 2
* record lock 확인
```sql
update t1 set c2 = 999 where c1 = 20;
```
```sql
-- blocked되어 update 수행 지연, record lock 걸림
```
* gap lock 확인
```sql
insert into t1 values (15, 150);
```
```sql
-- blocked되어 insert 수행 지연, gap lock 걸림
```
## 결과
* `Unique index`의 경우, 특정 row에 대한 `record lock`만 걸림
* `Non-unique index`의 경우, `next-key lock`이 걸림
    * record lock과 함께 해당 row 주변의 gap에도 gap lock이 걸림
    * next-key lock = record lock + gap lock
<br><br>

# Index 종류 별 lock 전파 확인
## secondary index
* secondary lock이 걸릴 경우 clustered index에도 lock이 함께 설정되는지 확인해보자
    * `Index 종류 별 lock 범위 비교` 실험에서 확인한 결과
        * secondary index가 unique → record lock
        * secondary index가 non unique → next-key lock
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
* non-unique secondary index(`idx1`)를 사용하여 next-key lock 발생
### session 1
```sql
-- 실험 준비
create table t1(c1 int primary key, c2 int, c3 int, index idx1(c2));
insert into t1 values (10, 100, 1000), (20, 200, 2000), (30, 300, 3000);

-- 실험 시작
set autocommit = 0;

start transaction;
select * from t1 where c2 = 200 for update;
-- secondary index 컬럼인 c2를 기준으로 c2=200 주변에 next-key lock이 걸림 : (100, 200]
-- clustered index에도 lock이 함께 설정된다면 c1=(10, 20]에도 lock이 걸릴 것이라고 예상
```
### session 2
* record lock 확인
```sql
update t1 set c3 = 9999 where c1 = 20;
```
```sql
-- blocked되어 update 수행 지연
-- clustered index에도 record lock이 함께 설정됨
```
* gap lock 확인
```sql
insert into t1 values (15, 400, 4000);
-- 의도적으로 c2에 걸린 gap lock에 포함되지 않은 데이터 삽입 (c2=400)
```
```sql
Query OK, 1 row affected (0.00 sec)
-- insert 성공
-- clustered index의 gap에는 lock이 설정되지 않음
```
## covering index
* covering index는 데이터 접근 시 테이블을 건드리지 않고 인덱스에서 처리
* 실제 row(clustered index)에도 lock이 함께 설정되는지 확인해보자
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
* non-unique인 covering index(`idx1`)를 사용하여 next-key lock 발생
### session 1
```sql
-- 실험 준비
create table t1(c1 int primary key, c2 int, c3 int, index idx1(c2, c3));
insert into t1 values (10, 100, 1000), (20, 200, 2000), (30, 300, 3000);

-- 실험 시작
set autocommit = 0;

start transaction;
select c2, c3 from t1 where c2 = 200 for update;
-- secondary index 컬럼인 c2를 기준으로 c2=200 주변에 next-key lock이 걸림 : (100, 200]
-- clustered index에도 lock이 함께 설정된다면 c1=(10, 20]에도 lock이 걸릴 것이라고 예상
explain select c2, c3 from t1 where c2 = 200 for update;
```
```sql
type            : ref
possible_keys   : idx1 
key             : idx1
key_len         : 5
Extra           : Using index -- covering index 사용
```
### session 2
* record lock 확인
```sql
update t1 set c2 = 999 where c1 = 20;
```
```sql
-- blocked되어 update 수행 지연, record lock 걸림
-- clustered index에도 record lock이 함께 설정됨
```
* gap lock 확인 - c1 (clustered index)
```sql
insert into t1 values (15, 400, 4000);
```
```sql
Query OK, 1 row affected (0.00 sec)
-- insert 성공
-- clustered index의 gap에는 lock이 설정되지 않음
```
* gap lock 확인 - c2 (covering index 컬럼, 조건 컬럼)
```sql
insert into t1 values (40, 250, 4000);
```
```sql
-- blocked되어 insert 수행 지연
-- 검색 조건(c2=200)을 기준으로 형성된 인덱스 범위 (100, 200]에 대해 gap lock이 적용됨
```
* gap lock 확인 - c3 (covering index 컬럼)
```sql
insert into t1 values (40, 400, 1500);
```
```sql
Query OK, 1 row affected (0.00 sec)
-- insert 성공
-- covering index 컬럼이지만 조건 컬럼에 포함되지 않으므로 lock 범위 결정에 영향을 주지 않음
```
## 결과
* secondary index, covering index
    * record lock : clustered index에 함께 설정됨
    * gap lock : clustered index에 설정되지 않음
## 결론
secondary index 또는 covering index를 사용하여 row를 검색하는 경우, clustered index에는 record lock만 설정되고 gap lock은 적용되지 않는 것을 확인할 수 있다.

InnoDB는 실제 데이터를 보호하기 위해 clustered index에 record lock만 설정하며, gap lock은 검색에 사용된 인덱스를 기준으로 형성된 범위(`(100, 200]`)에 대해서만 적용된다. 즉, gap lock은 검색에 사용된 인덱스 범위에 대해서만 적용되며 clustered index의 gap까지는 확장되지 않는다.

또한, 다중 컬럼 인덱스의 경우에도 lock 범위는 인덱스 전체가 아니라 where 조건에 사용된 인덱스 prefix를 기준으로 결정되며, 조건에 포함되지 않은 컬럼은 lock 범위 결정에 영향을 주지 않는다.
## covering index - 추가 실험
* 이전 실험에서 다중 컬럼 인덱스의 경우, lock 범위는 where 조건에 사용된 인덱스 `prefix`를 기준으로 결정됨을 확인
* 인덱스 prefix 조건을 만족하지 못하는 경우 locking 동작이 어떻게 달라지는지 확인해보자
### session 1
```sql
create table t1(c1 int primary key, c2 int, c3 int, index idx1(c2, c3));
insert into t1 values (10, 100, 1000), (20, 200, 2000), (30, 300, 3000);

set autocommit = 0;

start transaction;
select c2, c3 from t1 where c3 = 2000 for update;
explain select c2, c3 from t1 where c3 = 2000 for update;
explain analyze select c2, c3 from t1 where c3 = 2000 for update;
```
```sql
type            : index -- index full scan
possible_keys   : idx1 
key             : idx1
key_len         : 10
Extra           : Using where; Using index -- covering index를 사용하지만 where에서 조건 필터링 (prefix 깨짐)
```
```sql
| -> Filter: (t1.c3 = 2000)  (cost=0.55 rows=1) (actual time=0.031..0.0353 rows=1 loops=1)
    -> Covering index scan on t1 using idx1  (cost=0.55 rows=3) (actual time=0.0264..0.033 rows=3 loops=1)
 |
```
### session 2
* gap lock 확인
```sql
insert into t1 values (40, 250, 4000); -- c2의 blocking 확인
insert into t1 values (40, 400, 1500); -- c3의 blocking 확인
```
```sql
-- 둘다 blocked되어 insert 수행 지연
```
## 결과
* index full scan 수행
* scan된 인덱스 전체 범위에 대해 lock 설정
## 결론
인덱스 prefix를 만족하지 못할 경우, index full scan을 수행하게 되고 결과적으로 특정 범위가 아닌 scan된 인덱스 전체 범위에 lock이 설정된다. 

즉, 다중 컬럼 인덱스에서 lock 범위는 단순히 where 조건에 등장한 컬럼이 아니라 `실제로 사용된 인덱스 prefix`를 기준으로 결정된다.