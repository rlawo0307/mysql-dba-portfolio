# Lock(락)이란?
* 여러 트랜잭션이 동시에 같은 데이터를 접근할 때 데이터의 일관성을 보장하기 위해 접근을 제어하는 메커니즘
* InnoDB는 `row-level locking`을 지원
    * 테이블 전체가 아닌 개별 row 단위로 lock을 거는 방식
    * 실제로는 row가 아닌 index record를 lock
## Lock 상태 [→ lock 상태 확인 실습](labs/lock-monitoring.md)
```text
lock 요청 → 충돌 없음 → Granted

lock 요청 → 충돌 발생 → Waiting → Granted / Timeout / Deadlock
```
* `Granted`
    * lock이 정상적으로 획득된 상태
    * 트랜잭션이 lock을 얻어 해당 row에 대해 작업을 수행할 수 있는 상태
    * 다른 트랜잭션이 같은 row에 접근하면 대기 상태(waiting)로 전환
* `Waiting`
    * 트랜잭션이 lock을 요청했지만 다른 트랜잭션이 이미 보유 중인 상태
    * 대기 상태는 다음 중 하나로 종료됨
        * lock 획득(granted)
        * 대기 시간 초과 (lock wait timeout)
        * deadlock 감지 (deadlock detection) 
* `Lock Wait Timeout`
    * 설정된 대기 시간 초과
    * `innodb_lock_wait_timeout`으로 대기 시간 설정 가능
    * 1205 에러 발생
        * ERROR 1205 (HY000): Lock wait timeout exceeded
* `Deadlock Detection` [→ deadlock에 대한 내용 자세히 보기](deadlock.md)
    * deadlock 발생
    * 1213 에러 발생
        * ERROR 1213 (40001): Deadlock found
    * InnoDB가 rollback cost가 가장 적은 트랜잭션을 선택하여 rollback
## Lock 종류
* `Shared Lock`(S Lock)
    * 읽기 전용 lock
    * 여러 트랜잭션이 동시에 획득 가능
        * 일반 select는 MVCC snapshot로 읽기 가능
        * 동일한 shared lock은 동시에 획득 가능
        * 동시 locking write 대기
```sql
-- row level shared lock
select * from t1 where c1 = 10 for share; -- locking read
select * from t1 where c1 = 10 lock in share mode; -- locking read

-- table-level shared lock
lock tables t1 read;
```
* `Exclusive Lock`(X Lock)
    * 쓰기 전용 lock
    * 한 트랜잭션만 해당 데이터를 수정할 수 있도록 보장
        * 일반 select는 MVCC snapshot로 읽기 가능
        * 동시 locking read 대기
        * 동시 locking write 대기
    * update/delete 쿼리는 내부적으로 write lock이 자동으로 걸림
```sql
-- row level write lock 
select * from t1 where c1 = 1 for update; -- locking read

-- table level write lock
lock tables t1 write;
```

<center>

|                  |설명    |일반 select(MVCC)|동시 locking read|동시 write|
|------------------|:------:|:--------------:|:-:|:-:|
|**Shared Lock**   |읽기 전용|가능|가능|대기|
|**Exclusive Lock**|쓰기 전용|가능|대기|대기|
</center>

## Locking Read
* 일반적인 select는 MVCC를 통해 lock을 획득하지 않음(`Consistent Read`)
* 특정 상황에서 읽으면서 lock을 획득하고자 하는 요구가 있을 수 있음(`Current Read`)
    * '내가 읽은 데이터가 트랜잭션 동안 수정되지 않게 하고 싶다'
* [→ consistent read와 current read 비교에 대한 내용 자세히 보기](consistent-vs-current-read.md)
```sql
-- shared lock 획득
select * from t1 where c1 = 10 for share;
select * from t1 where c1 = 10 lock in share mode;

-- write lock 획득
select * from t1 where c1 = 1 for update;
```
<br><br>

# InnoDB Lock 범위 메커니즘
* InnoDB의 lock은 row 단위가 아닌 index record 기준으로 동작 [→ 인덱스에 따른 lock 범위 비교 실습](labs/locking-index-behavior.md)
* InnoDB는 phantom read를 방지하기 위해 다양한 lock 메커니즘을 사용
    * phantom read는 범위 조건 쿼리에서 자주 발생
    * 단순히 특정 row를 잠그는 것이 아닌, index 구조를 기반으로 lock 범위를 결정하여 phantom read를 방지
* InnoDB index page 내부에는 page 범위를 표현하기 위한 가상의 레코드가 존재
    * `Infimum pseudo-record`
        * 현재 page의 가장 작은 값보다 작은 위치를 나타냄
        * 개념적으로 `-∞`에 해당
    * `Supremum pseudo-record`
        * 현재 page의 가장 큰 값보다 더 큰 위치를 나타냄
        * 개념적으로 `∞`에 해당
* 종류
    * table level
        * `Intention Lock`(IS, IX)
    * index level
        * `Record Lock`
        * `Gap Lock`
        * `Next-Key Lock`
        * `Insert Intention Lock`
### Intention Lock
(이해하고 다시 정리하기)
<!--
?????????????????????
이해하고 다시 정리하기!

* row-level lock을 걸기 전에 table에 의도를 표시하는 lock
    * '이 트랜잭션은 이 테이블의 row에 lock을 걸 예정이다'
* 사용자가 직접 설정하지 않음
    * InnoDB가 자동으로 관리
    * row lock이 설정되면 해당 테이블에는 자동으로 intention lock이 설정됨
* 테이블에 lock이 걸려있는지 table-level에서 빠르게 판단할 수 있음
* 종류
    * `Intention Shared`(IS)
        * row에 shared lock을 걸 예정임을 표시
    * `Intention Exclusive`(IX)
        * row에 exclusive lock을 걸 예정임을 표시
```
-->
### Record Lock
* 특정 index record에만 lock을 거는 방식
* lock 범위 : `특정 인덱스 키 하나`
* phantom read 방지 기능 없음
* 주로 unique index equality 조건에서 사용
    * 단일 record에만 접근하기 때문에 range lock이 필요 없음
```sql
select * from t1 where c1 = 10 for update;
```
### Gap Lock
* index 범위(gap)에 lock을 거는 방식
* lock 범위 : `(이전 인덱스 키, 특정 인덱스 키)`
* lock된 범위 내에는 다른 트랜잭션이 데이터를 insert할 수 없음
* 주로 next-key lock의 일부로 사용됨
* range 조건 결과 row가 없는 경우 gap lock만 설정될 수 있음
```sql
select * from t1 where c1 between 10 and 20 for update;
-- 일반적으로 range 조건은 next-key lock이 걸림
-- 하지만 range 조건 결과 row가 없는 경우 gap lock만 설정될 수 있음
```
* 하지만 gap lock은 특정 인덱스 key를 보호하지 않음
    * (10, 15)에 lock이 걸려있을 경우
    * 데이터 12를 insert하는 것은 불가능
    * 데이터 15를 update하는 것은 가능
    * 즉, 특정 record 자체를 보호하지 못하므로 next-key lock이 필요
### Next-Key Lock
* Record Lock + Gap Lock
    * 새로운 종류의 lock이 아님
    * record lock과 gap lock을 동시에 적용하는 방식
* 특정 index record와 앞쪽 gap에 lock을 거는 방식
* lock 범위 : `(이전 인덱스 키, 특정 인덱스 키]`
* **InnoDB에서 range scan 시 기본적으로 사용되는 locking 방식**
    * **phantom read를 방지하기 위해 사용**
```sql
select * from t1 where c1 between 10 and 20 for update;
-- 일반적으로 next-key lock이 걸림
-- 하지만 특정 상황에서 gap lock이 걸릴 수도 있음
```
* range scan 과정에서 여러 개의 next-key lock이 연속적으로 설정될 수 있음
    * index record마다 하나씩 lock이 생성
    * range 조건 범위가 클수록 생성되는 lock이 많아져 락 경합이 발생할 수 있음
```sql
ex) 인덱스 구조 : [5, 10, 15, 20, 25]
    range 조건 : where c1 between 10 and 20

    scan되는 record : 10 → 15 → 20
    next-key lock : (5 , 10] 
                    (10 , 15]
                    (15 , 20]
------------------------------------------------
ex) 인덱스 구조 : [5, 10, 15, 20, 25]
    range 조건 : where c1 > 25

    scan되는 record : 없음
    next-key lock : (25, Supremum(∞)) -- Supremum은 실제 record가 아니므로 gap lock만 걸림
------------------------------------------------
ex) 인덱스 구조 : [5, 10, 15, 20, 25]
    range 조건 : where c1 < 5

    scan되는 record : 없음
    next-key lock : (Infimum(-∞), 5) -- Infimum은 실제 record가 아니므로 gap lock만 걸림
```
### Insert Intention Lock
* gap에 데이터를 insert하려는 의도를 나타내는 lock
* 여러 트랜잭션이 같은 gap에 동시에 insert하려고 할 때 불필요한 blocking을 방지
* 서로 다른 insert intention lock끼리는 충돌하지 않음
    * 같은 gap에 동시에 데이터 insert 가능
* range locking(gap lock/next-key lock)과는 충돌
    * 이미 다른 트랜잭션이 특정 gap에 range locking을 설정했다면
    * 해당 gap에 대한 insert는 lock이 해제될 때까지 대기
<br><br>

# Lock과 Index의 관계
* InnoDB에서 lock은 흔히 `row-level lock`이라고 부르지만, 실제로는 row 자체가 아닌 `index record 기준`으로 lock이 설정됨
    * row locking (X)
    * index record locking (O)
* lock 범위는 쿼리 조건이 아니라 index scan 결과에 따라 달라짐
### 인덱스가 있는 경우
* 조건 컬럼에 인덱스가 존재하면 index scan을 수행
* scan된 index record 기준으로 lock 설정
* 인덱스가 존재하면 lock의 범위를 좁힐 수 있음
### 인덱스가 없는 경우
* 조건 컬럼에 인덱스가 없다면 full table scan을 수행
    * clustered index를 순차적으로 scan
    * scan된 모든 clusterd index record에 대해 record lock이 걸림
    * 결과적으로 table 전체가 잠긴 것과 유사하게 동작
    * 즉, row-level lock이지만 `table-level lock`처럼 동작
* 인덱스가 없다면 lock 범위가 불필요하게 넓어질 수 있음
    * 다른 트랜잭션의 update/insert가 광범위하게 blocking될 수 있음
    * 실무에서는 locking read 조건 컬럼에 인덱스를 두는 것이 중요