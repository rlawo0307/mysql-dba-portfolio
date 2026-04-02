# Consistent Read vs Current Read 비교
* MySQL의 설계 철학
    * 읽기는 최대한 자유롭게 → `Consistent Read`(MVCC)
    * 쓰기는 확실하게 통제 → `Current Read`(lock)
<br><br>

# Consistent Read
* [MVCC](transaction-mechanism.md)를 이용해서 과거의 데이터(snapshot)을 읽는 방식
* 일반 select 수행 시 동작
* 내부 동작
```
    1. 현재 row에서 trx_id 확인
    2. 내가 볼 수 없는 버전이면 undo log를 따라감
    3. 볼 수 있는 과거 버전을 찾음
    4. snapshot 생성
```
* lock을 사용하지 않음
* 다른 트랜잭션을 막지 않음
* isolation level은 consistent read의 동작만 바꿈
    * 어떤 snapshot을 보게 할지에 관여
    * **Serializable에서는 일반 select도 lock을 획득하지만 current read가 아님**
    * Serializable = consistent read + shared lock
<br>

# Current Read
* 항상 최신 데이터를 읽고, 동시에 lock을 획득하는 방식
* [locking read](locking.md), update/delete 수행 시 동작
    * update/delete는 내부적으로 조건에 맞는 row를 찾을 때 current read를 사용하며 동시에 write lock을 획득
* 내부 동작
```
    1. 현재 row를 직접 읽음
    2. lock 획득
    3. 다른 트랜잭션 접근 제한
```
* snapshot을 생성하지 않음(undo log를 따라가지 않음)
<br><br>

# 요약
<center>

|                    |읽는 시점|MVCC|undo log|lock|대표 쿼리|특징|
|--------------------|:------:|:-:|:-------:|:-:|:------:|:--:|
|**Consistent Read**|과거 snapshot|사용|따라감|미사용|일반 select|non-blocking|
|**Current Read**  |현재 row|미사용|따라가지 않음|사용|for update<br>for share<br>lock in share mode<br>update / delete<br>|blocking 가능|
</center>

* InnoDB에서 데이터 읽기 방식은 consistent read와 current read로 나뉨
* 데이터 읽기 방식은 쿼리 종류에 의해 결정됨
    * consistent read는 MVCC를 이용해 snapshot을 읽으며 lock을 사용하지 않음
    * current read는 항상 최신 데이터를 읽으며 lock을 획득
* isolation level은 consistent read의 동작을 결정
    * 어떤 snapshot을 보게 할지에 관여
* current read는 isolation level과 관계없이 항상 동일하게 동작