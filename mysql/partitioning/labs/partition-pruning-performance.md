# 목차
* partition pruning 성능 실험
    * 일반 테이블 vs partitioned table
* 참고 자료
    * [→ Partition Pruning이란?](../partitioning-basics.md)
    * [→ explain 결과 해석](../../optimization/explain.md)
<br><br>

# Partition Pruning 성능 실험
* partition pruning은 쿼리 조건에 따라 필요한 partition만 선택적으로 조회하는 최적화 기법으로 불필요한 partition을 읽지 않아 I/O가 감소하여 성능을 향상시킨다
* 즉, 파티셔닝의 성능 향상은 대부분 pruning에 의해 발생한다
* 여러 상황에서 pruning 적용 유무에 따른 성능 차이를 비교해보자
<br><br>

# 일반 테이블 vs Partitioned table
* pruning이 발생하지 않는 일반 테이블과 pruning이 발생하는 partitioned table의 성능을 비교해보자
## 일반 테이블
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n from seq;
```
```sql
explain select * from t1 where c1 >= 200000 and c1 < 300000;
explain analyze select * from t1 where c1 >= 200000 and c1 < 300000;
```
```sql
table           : t1
partitions      : NULL -- 모든 파티션을 탐색
type            : ALL -- full table scan
rows            : 499550
Extra           : Using where -- where 절에서 조건 필터링
```
```sql
| -> Filter: ((t1.c1 >= 200000) and (t1.c1 < 300000))  (cost=50203 rows=55494) (actual time=78.4..206 rows=100000 loops=1)
    -> Table scan on t1  (cost=50203 rows=499550) (actual time=0.0289..176 rows=500000 loops=1)
 |
```
## Partitioned table
```sql
set session cte_max_recursion_depth = 500000;

create table t2(c1 int) partition by range(c1)(
    partition p1 values less than(100000),
    partition p2 values less than(200000),
    partition p3 values less than(300000),
    partition p4 values less than(400000),
    partition p5 values less than(500000),
    partition pmax values less than(maxvalue)
);

insert into t2 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n from seq;
```
```sql
explain select * from t2 where c1 >= 200000 and c1 < 300000;
explain analyze select * from t2 where c1 >= 200000 and c1 < 300000;
```
```sql
table           : t2
partitions      : p3
type            : ALL -- full table scan
rows            : 96895
Extra           : Using where -- where 절에서 조건 필터링
```
```sql
| -> Filter: ((t2.c1 >= 200000) and (t2.c1 < 300000))  (cost=9746 rows=10764) (actual time=0.0374..51.1 rows=100000 loops=1)
    -> Table scan on t2  (cost=9746 rows=96895) (actual time=0.0356..42.2 rows=100000 loops=1)
 |
```
## 결과
* 일반 테이블
    * 모든 파티션을 탐색
    * 스캔 예상 row : 499550 (거의 전체 row)
    * 실행 시간 : `78.4..206`
* partitioned table
    * p3 파티션만 탐색
    * 스캔 예상 row : 96895 (약 1/5)
    * 실행 시간 : `0.0374..51.1`
## 결론
실험 결과, partition pruning이 발생하는 경우와 발생하지 않는 경우의 성능 차이가 명확하게 나타났다.

일반 테이블의 경우, 전체 데이터를 대상으로 full table scan이 발생하여 약 50만 건의 데이터를 모두 탐색했다. 반면, partitioned table의 경우 조건에 해당하는 파티션(p3)만 선택적으로 탐색하여 약 1/5 수준의 데이터만 읽었다. 이로 인해 실행 시간에서도 큰 차이가 발생했으며 partitioned table일 경우 실행 시간이 약 4배 수준으로 단축되었다.

파티셔닝의 성능 향상은 단순히 테이블을 나누는 것에서 오는 것이 아니라, partition pruning을 통해 불필요한 데이터 접근을 줄이는 것에서 발생한다. 즉, 파티셔닝 자체가 성능을 보장하는 것이 아니며 pruning이 발생하지 않으면 오히려 성능 이점이 사라질 수 있다. pruning이 적용되면 읽는 데이터 범위가 감소하여 I/O가 줄어들고, 이는 실행 시간 단축으로 이어진다.

따라서, 파티셔닝을 효과적으로 사용하기 위해서는 partition key를 기준으로 조회하는 쿼리를 작성해야 하며, 옵티마이저가 partition 범위를 계산할 수 있도록 해야 한다.
<br>