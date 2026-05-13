# Consistent Read vs Current Read 비교
* MySQL의 설계 철학
    * 읽기는 최대한 자유롭게 → `Consistent Read` (MVCC)
    * 쓰기는 확실하게 통제 → `Current Read` (lock)
<br>
<center>

|                    |읽는 시점|MVCC|undo log|lock|대표 쿼리|특징|비고|
|--------------------|:------:|:-:|:-------:|:-:|:------:|:--:|:---:|
|**Consistent Read**|snapshot 기준|사용|필요 시 따라감|미사용|일반 select|non-blocking|serializable은 shared lock 추가|
|**Current Read**  |현재 row|미사용|snapshot<br>조회에 미사용|사용|for update<br>for share<br>lock in share mode<br>update / delete<br>|blocking 가능||
</center>
<br><br>

# Consistent Read
* 특정 시점을 기준으로 일관된 데이터를 읽는 snapshot read
    * MySQL InnoDB에서는 [MVCC](transaction-mechanism.md)와 read view를 이용해 구현됨
* lock을 사용하지 않으므로 다른 트랜잭션을 막지 않음
* [isolation level](isolation-levels.md)은 consistent read의 동작에 직접적인 영향을 줌
    * 언제 read view를 생성하고, 어떤 데이터를 보고, 재사용할지에 관여
    * 단, Serializable의 경우 consistent read 계열이지만 일반 select라도 shared lock을 획득함
        * `Serializable = consistent read + shared lock`
        * 따라서, 일반 select라도 non-blocking이 아닐 수 있으며 다른 트랜잭션 때문에 대기할 수 있음
### 동작 방식
* 일반 select에서 동작
* serializable에서는 shared lock이 추가됨
```
1. read view 생성 (또는 기존 read view 재사용)
2. 현재 row의 trx_id 확인
3. visibility 판단
    - visible 하다면 현재 row를 반환
    - visible 하지 않으면 undo log를 따라 과거 버전 탐색 후 반환
```
<br>

# Current Read
* 항상 최신 데이터를 읽고, 동시에 lock을 획득하는 방식
* read view를 생성하지 않음
* read view visibility 판단을 위해 undo log를 따라 과거 버전을 조회하지 않음
### 동작 방식
* [locking read](locking.md), update/delete 수행 시 동작
    * update/delete는 내부적으로 조건에 맞는 row를 찾을 때 current read를 사용하며 동시에 write lock을 획득
```
1. 최신 row를 기준으로 조회
2. lock 획득
3. 다른 트랜잭션 접근 제한
```
<br>