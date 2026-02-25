# 목차
* 인덱스 구성에 따른 성능 비교
    * 단일 인덱스
    * 복합 인덱스
    * Index Merge
* 참고 자료
    * [→ explain에 대한 자세한 내용 보기](../../optimization/explain.md)
    * [→ index 종류에 대한 자세한 내용 보기](../index-basics.md)
    * [→ 각 scan 방식에 대한 개념 자세히 보기](../../index/index-basics.md)
<br>

# 인덱스 구성에 따른 성능 비교
## AND로 연결된 조건
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int, c3 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;
```
### 단일 인덱스가 여러개 존재
* MySQL은 기본적으로 한 테이블에 대해 하나의 인덱스만을 사용
* 옵티마이저는 존재하는 인덱스의 cost를 비교하여 더 좋은 인덱스를 선택
```sql
create index idx1 on t1(c1);
create index idx2 on t1(c2);
analyze table t1;

explain select /*+ no_index_merge(t1)*/ * from t1 where c1 = 10 and c2 = 50;
explain analyze select /*+ no_index_merge(t1)*/ * from t1 where c1 = 10 and c2 = 50;
```
```sql
type            : ref
possible_keys   : idx1, idx2
key             : idx1
key_len         : 5
rows            : 487
filtered        : 0.10
Extra           : Using where
```
```sql
| -> Filter: (t1.c2 = 50)  (cost=122 rows=0.488) (actual time=0.86..1.48 rows=1 loops=1)
    -> Index lookup on t1 using idx1 (c1=10)  (cost=122 rows=487) (actual time=0.0244..1.46 rows=487 loops=1)
 |
```
### 복합 인덱스
```sql
create index idx3 on t1(c1, c2);
analyze table t1;

explain select * from t1 where c1 = 10 and c2 = 50;
explain analyze select * from t1 where c1 = 10 and c2 = 50;

drop index idx3 on t1;
```
```sql
type            : ref
possible_keys   : idx1, idx2, idx3
key             : idx3
key_len         : 10
rows            : 1
filtered        : 100
Extra           : NULL
```
```sql
| -> Index lookup on t1 using idx3 (c1=10, c2=50)  (cost=0.35 rows=1) (actual time=0.0275..0.0319 rows=1 loops=1)
 |
```
### Index Merge
```sql
explain select /*+ index_merge(t1 idx1, idx2)*/ * from t1 where c1 = 10 and c2 = 50;
explain analyze select /*+ index_merge(t1 idx1, idx2)*/ * from t1 where c1 = 10 and c2 = 50;
```
```sql
type            : index_merge
possible_keys   : idx1, idx2
key             : idx1, idx2
key_len         : 5,5
rows            : 1
filtered        : 100
Extra           : Using intersect(idx1, idx2), Using where
```
```sql
| -> Filter: ((t1.c2 = 50) and (t1.c1 = 10))  (cost=3.42 rows=1) (actual time=0.321..0.529 rows=1 loops=1)
    -> Intersect rows sorted by row ID  (cost=3.42 rows=1) (actual time=0.32..0.528 rows=1 loops=1)
        -> Index range scan on t1 using idx1 over (c1 = 10)  (cost=1.65 rows=487) (actual time=0.0647..0.267 rows=487 loops=1)
        -> Index range scan on t1 using idx2 over (c2 = 50)  (cost=1.67 rows=500) (actual time=0.00784..0.214 rows=500 loops=1)
 |
```
### 실행 시간 비교
* 단일 인덱스 : `0.86..1.48`
* 복합 인덱스 : `0.0275..0.0319`
* Index Merge : `0.321..0.529`
### 결론
and 연산으로 연결된 조건에서 실험한 결과, 복합 인덱스 → Index Merge → 단일 인덱스 순서대로 빠른 실행 시간을 보인다.

복합 인덱스는 두 조건을 하나의 B+Tree 탐색으로 동시에 처리하기 때문에 추가적인 row filtering이나 rowid 계산 과정 없이 원하는 row에 직접 접근할 수 있다. 그 결과, I/O와 CPU 연산이 최소화되어 가장 빠른 성능을 보인다.
Index Merge는 두 개의 단일 인덱스를 각각 탐색한 후, rowid를 기준으로 교집합을 계산하여 결과를 도출한다. 이 방식은 단일 인덱스보다 불필요한 row 접근은 줄였지만 추가적인 과정에서 발생한 비용으로 인해 복합 인덱스보다는 느린 결과를 보인다.
단일 인덱스는 하나의 조건만 인덱스로 처리하고 나머지 조건은 레코드 접근 후 필터링을 수행한다. 따라서 많은 레코드를 읽어야 하며 그만큼 I/O와 CPU 연산 비용이 증가하여 가장 낮은 성능을 보인다.
## OR로 연결된 조건
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int, c3 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;

create index idx1 on t1(c1);
create index idx2 on t1(c2);
create index idx3 on t1(c1, c2);
analyze table t1;
```
### 단일/복합 인덱스 존재 + Index Merge 제한
```sql
explain select /*+ no_index_merge(t1)*/ * from t1 where c1 = 10 or c2 = 50;
explain analyze select /*+ no_index_merge(t1)*/ * from t1 where c1 = 10 or c2 = 50;
```
```sql
type            : ALL
possible_keys   : idx1, idx2, idx3
key             : NULL
key_len         : NULL
rows            : 499365
filtered        : 0.20
Extra           : Using where
```
```sql
| -> Filter: ((t1.c1 = 10) or (t1.c2 = 50))  (cost=50249 rows=1006) (actual time=0.0892..232 rows=983 loops=1)
    -> Table scan on t1  (cost=50249 rows=499365) (actual time=0.0168..197 rows=500000 loops=1)
 |
```
### Index Merge
```sql
explain select * from t1 where c1 = 10 or c2 = 50;
explain analyze select * from t1 where c1 = 10 or c2 = 50;
```
```sql
type            : index_merge
possible_keys   : idx1, idx2, idx3
key             : idx1, idx2
key_len         : 5,5
rows            : 983
filtered        : 100
Extra           : Using union(idx1, idx2), Using where
```
```sql
| -> Filter: ((t1.c1 = 10) or (t1.c2 = 50))  (cost=472 rows=983) (actual time=0.123..5.79 rows=983 loops=1)
    -> Deduplicate rows sorted by row ID  (cost=472 rows=983) (actual time=0.121..5.69 rows=983 loops=1)
        -> Index range scan on t1 using idx1 over (c1 = 10)  (cost=53.6 rows=519) (actual time=0.096..0.359 rows=519 loops=1)
        -> Index range scan on t1 using idx2 over (c2 = 50)  (cost=48 rows=464) (actual time=0.00821..0.245 rows=464 loops=1)
 |
```
### 결과 비교
* 단일 인덱스, 복합 인덱스
    * 인덱스가 존재하지만 full table scan으로 풀림
    * 실행 시간 : `0.0892..232`
* Index Merge
    * 실행 시간 : `0.123..5.79`
### 결론
or 조건에서 실험해본 결과, MySQL 옵티마이저는 단일 인덱스나 복합 인덱스를 사용하지 않고 full table scan을 선택했다. 이는 or 조건 특성 상, 한 번의 탐색으로 모든 조건을 만족하는 범위를 정의하기 어렵기 때문이다. 즉, or 연산으로 연결된 두 조건을 동시에 처리할 수 없기 때문에 복합 인덱스가 효율적으로 사용되지 않는 경우가 많다. 또한, 단일 인덱스를 사용하더라도 다른 조건을 처리하기 위해 많은 레코드를 추가로 읽어야 하므로 비용이 크게 증가한다. 이로 인해 옵티마이저는 table 또는 index full scan이 더 cost가 낮다고 판단할 수 있다.

실행 시간을 비교해 보면, full table scan보다 Index Merge가 월등히 빠른 것을 확인할 수 있다.

즉, or 조건에서는 복합 인덱스보다 단일 인덱스를 기반으로 한 Index Merge 전략이 더 좋은 성능을 보일 수 있다. 단, 데이터 분포와 조건의 selectivity에 따라 성능이 달라질 수 있으므로 실제 환경에서는 실행 계획과 통계 정보를 함께 고려한 테이블 및 인덱스 설계가 필요하다.