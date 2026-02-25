# 목차
* 인덱스 별 조회 성능 차이 분석
    * Clustered Index
    * Secondary Index
    * Covering Index
* 참고자료
<br>

# 인덱스 별 조회 성능 차이 분석
## 단일 row를 조회할 경우
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n from seq;
```
### Clustered Index
```sql
alter table t1 add primary key (c1);
analyze table t1;

explain select * from t1 where c1 = 10;
explain analyze select * from t1 where c1 = 10;
```
```sql
type            : const
possible_keys   : PRIMARY
key             : PRIMARY
key_len         : 4 -- pk는 자동으로 not null. null flag가 붙지 않음
rows            : 1
filtered        : 100.00
Extra           : NULL
```
```sql
| -> Rows fetched before execution  (cost=0..0 rows=1) (actual time=40e-6..80e-6 rows=1 loops=1)
 |
```
### Secondary Index
* **테이블 t1을 삭제하고 다시 생성해야 함**
    * Clustered Index 실험에서 pk를 drop하면 hidden clustered index(row-d)pk가 자동으로 생성됨
    * 테이블의 물리적 저아 구조가 변경됨
    * 실험 조건이 달라짐
```sql
drop table t1;
create table t1 (c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n from seq;

create index idx on t1(c1);
analyze table t1;

explain select * from t1 where c1 = 10;
explain analyze select * from t1 where c1 = 10;
drop index idx on t1;
```
```sql
type            : ref
possible_keys   : idx
key             : idx
key_len         : 5
rows            : 1
filtered        : 100.00
Extra           : NULL
```
```sql
| -> Index lookup on t1 using idx (c1=10)  (cost=0.35 rows=1) (actual time=0.102..0.105 rows=1 loops=1)
 |
```
### Covering Index
```sql
create index idx on t1(c1, c2);
analyze table t1;

explain select * from t1 where c1 = 10;
explain analyze select * from t1 where c1 = 10;
drop index idx on t1;
```
```sql
type            : ref
possible_keys   : idx
key             : idx
key_len         : 5
rows            : 1
filtered        : 100.00
Extra           : Using index -- covering index 사용
```
```sql
| -> Covering index lookup on t1 using idx (c1=10)  (cost=1.1 rows=1) (actual time=0.0166..0.0202 rows=1 loops=1)
 |
```
### 실행 시간 비교
* Clustered Index : `40e-6..80e-6`
* Secondary Index : `0.102..0.105`
* Covering Index : `0.0166..0.0202`
### 결론
같은 테이블에서 동일한 조건으로 인덱스 구조만 변경하여 실험한 결과, Clustered Index, Covering Index, Secondary Index 순서대로 빠른 실행결과를 보인다. 

Clustered Index는 pk 기반 단일 B+Tree 탐색만 수행하므로 가장 단순한 접근 경로를 가진다.Covering Index는 Secondary Index만 탐색하며, 쿼리에 필요한 모든 컬럼이 Secondary Index leaf 노드에 포함되어 있어 추갖거인 back lookup이 발생하지 않는다. 마지막으로 Secondary Index는 Secondary Index 탐색 후 pk를 통한 back lookup 과정이 추가되므로 상대적으로 더 많은 B+Tree 탐색이 필요하다.

하지만 현재 실험은 단일 row만 조회하는 조건이므로 Secondary Index와 Covering Index 간 실행 시간 차이가 매우 미미하다. 더 명확한 차이를 확인하기 위해서는 추가적인 실험이 필요하다.
## 많은 row 조회 + 낮은 selectivity 조건 조회
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n from seq;
```
### Clustered Index
```sql
alter table t1 add primary key (c1);
analyze table t1;

explain select * from t1 where c1 < 250000;
explain analyze select * from t1 where c1 < 250000;
```
```sql
type            : range
possible_keys   : PRIMARY
key             : PRIMARY
key_len         : 4
rows            : 249780
filtered        : 100.00
Extra           : Using where
```
```sql
| -> Filter: (t1.c1 < 250000)  (cost=50197 rows=249780) (actual time=2.06..70.8 rows=249999 loops=1)
    -> Index range scan on t1 using PRIMARY over (c1 < 250000)  (cost=50197 rows=249780) (actual time=2.06..56.9 rows=249999 loops=1)
 |
 ```
### Secondary Index
* **테이블 t1을 삭제하고 다시 생성해야 함**
    * Clustered Index 실험에서 pk를 drop하면 hidden clustered index(row-d)pk가 자동으로 생성됨
    * 테이블의 물리적 저아 구조가 변경됨
    * 실험 조건이 달라짐
* 옵티마이저 판단에 의해 full table scan으로 풀릴 가능성이 있음
    * `force index` 힌트를 사용해 인덱스 사용 유도
```sql
drop table t1;
create table t1 (c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n from seq;

create index idx on t1(c1);
analyze table t1;

explain select * from t1 force index (idx) where c1 < 250000;
explain analyze select * from t1 force index (idx) where c1 < 250000;
drop index idx on t1;
```
```sql
type            : range
possible_keys   : idx
key             : idx
key_len         : 5
rows            : 1 -- force index 힌트로 인한 잘못된 추정값
filtered        : 100.00
Extra           : Using index condition
```
```sql
| -> Index range scan on t1 using idx over (NULL < c1 < 250000), with index condition: (t1.c1 < 250000)  (cost=0.71 rows=1) (actual time=0.0781..282 rows=249999 loops=1)
 |
```
### Covering Index
```sql
create index idx on t1(c1, c2);
analyze table t1;

explain select * from t1 where c1 < 250000;
explain analyze select * from t1 where c1 < 250000;
drop index idx on t1;
```
```sql
type            : range
possible_keys   : idx
key             : idx
key_len         : 5
rows            : 249603
filtered        : 100.00
Extra           : Using where; Using index
```
```sql
| -> Filter: (t1.c1 < 250000)  (cost=50408 rows=249603) (actual time=0.0203..124 rows=249999 loops=1)
    -> Covering index range scan on t1 using idx over (NULL < c1 < 250000)  (cost=50408 rows=249603) (actual time=0.0189..110 rows=249999 loops=1)
 |
```
### 실행 시간 비교
* Clustered Index : `2.06..70.8`
* Secondary Index : `0.0781..282`
* Covering Index : `0.0203..124`
### 결론
낮은 selectivity 조건으로 여러 row를 조회해 본 결과, Clustered Index가 가장 빠른 실행 시간을 보였으며, 특히 Secondary Index와 Covering Index 간의 실행 시간의 차이가 뚜렷하게 나타난다. 이는 다수의 row에 대해 Secondary Index의 back lookup이 반복되면서 Random I/O 비용이 누적되기 때문이다.