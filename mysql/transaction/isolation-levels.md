# 트랜잭션 이상 현상(Anomaly)
* 여러 트랜잭션이 동시에 실행될 때, 적절한 격리가 이루어지지 않으면 데이터의 일관성이 깨지는 현상이 발생할 수 있음
* 종류
    * `Dirty Read` : commit되지 않은 데이터를 다른 트랜잭션이 읽음
    * `Non-Repeatable Read` : 같은 row를 두 번 읽었는데 값이 달라짐
    * `Phantom Read` : 같은 조건으로 조회했는데 행의 개수가 달라짐
<br>

# Isolation Level
* 동시에 실행되는 트랜잭션들이 서로의 변경 사항을 어디까지 볼 수 있는지를 결정하는 기준
```
- 격리 수준 ↑ → 안정성 ↑, 동시성/성능 ↓
- 격리 수준 ↓ → 동시성/성능 ↑, 이상현상 가능성 ↑
```
* 종류
    * `READ UNCOMMITTED`
        * commit, rollback 여부와 상관없이 다른 트랜잭션에서 데이터 조회 가능
        * dirty read 가능
        * 성능은 좋지만 데이터 신뢰성이 매우 낮음
        * 실무에서는 거의 사용하지 않음
    * `READ COMMITTED`
        * commit이 완료된 데이터만 다른 트랜잭션에서 조회 가능
        * select 쿼리마다 새로운 Read View 생성 (InnoDB 기준)
            * 데이터를 읽을 때마다 결과가 바뀔 수 있음
        * dirty read 불가능
        * non-repeatable read, phantom read 가능
        * oracle, postgreSQL의 기본 isolation level
    * `REPEATABLE READ`
        * 같은 트랜잭션 내에서 동일한 select에 대해 항상 같은 결과를 보장
        * 트랜잭션 시작 후 처음 생성된 Read View를 끝까지 유지
        * dirty read, non-repeatable read 불가능
        * SQL 표준에서는 phantom read 가능
        * InnoDB에서는 Next-key Lock을 사용해서 phantom read 방지
        * **MySQL의 기본 isolation level**
    * `SERIALIZABLE`
        * 모든 트랜잭션을 직렬 실행할 것과 동일한 결과를 보장
        * 일반 select 쿼리도 lock을 획득하여 실행됨
        * 가장 단순하면서 가장 엄격한 격리 수준
        * 모든 이상현상 방지
        * 동시 처리 성능이 매우 떨어짐

<center>

|                    |Dirty Read|Non-Repeatable Read|Phantom Read|
|--------------------|:--------:|:-----------------:|:----------:|
|**READ UNCOMMITTED**|O|O|O|
|**READ COMMITTED**  |X|O|O|
|**REPEATABLE READ** |X|X|O (InnoDB의 경우 X)|
|**SERIALIZABLE**    |X|X|X|
</center>