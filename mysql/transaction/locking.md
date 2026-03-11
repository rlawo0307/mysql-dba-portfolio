# Lock(락)이란?
* 여러 트랜잭션이 동시에 같은 데이터를 접근할 때 데이터의 일관성을 보장하기 위해 접근을 제어하는 메커니즘
* InnoDB는 `row-level locking`을 지원
    * 테이블 전체가 아닌 개별 row 단위로 lock을 거는 방식
    * 실제로는 row가 아닌 index record를 lock
## Lock 종류
* `Shared Lock`(S Lock)
    * 읽기 전용 lock
    * 여러 트랜잭션이 동시에 획득 가능
        * 일반 select는 MVCC snapshot로 읽기 가능
        * 동시 locking read 가능
        * 동시 locking write 대기
* `Exclusive Lock`(X Lock)
    * 쓰기 전용 lock
    * 한 트랜잭션만 해당 데이터를 수정할 수 있도록 보장
        * 일반 select는 MVCC snapshot로 읽기 가능
        * 동시 locking read 대기
        * 동시 locking write 대기

<center>

|                  |설명    |일반 select(MVCC)|동시 locking read|동시 write|
|------------------|:------:|:--------------:|:-:|:-:|
|**Shared Lock**   |읽기 전용|가능|가능|대기|
|**Exclusive Lock**|쓰기 전용|가능|대기|대기|
</center>

## Locking Read
* 일반적인 select는 MVCC를 통해 lock을 획득하지 않음
* 특정 상황에서 읽으면서 lock을 획득하고자 하는 요구가 있을 수 있음
    * '내가 읽은 데이터가 트랜잭션 동안 수정되지 않게 하고 싶다'
* 대표적인 상황
    * `for update` : X lock 획득
    * `lock in share mode` : S lock 획득
<br><br>

# InnoDB Lock 범위 메커니즘
* InnoDB는 phantom read를 방지하기 위해 다양한 lock 메커니즘을 사용
    * phantom read는 범위 조건 쿼리에서 자주 발생
    * 단순히 특정 row를 잠그는 것이 아닌, index 구조를 기반으로 lock 범위를 결정하여 phantom read를 방지
## 종류
* `Record Lock`
    * 특정 index record에만 lock을 거는 방식
* `Gap Lock`
    * index 범위(gap)에 lock을 거는 방식
* `Next-Key Lock`
    * Record Lock + Gap Lock
    * 특정 index record와 앞쪽 gap에 lock을 거는 방식    
       