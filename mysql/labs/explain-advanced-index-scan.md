# 목차
* 고급 인덱스 스캔 (EXPLAIN)
    * Index Skip Scan
    * Loose Index Scan
* Index Merge vs 복합 인덱스
    * AND 조건 실행 계획 비교
    * OR 조건 실행 계획 비교
<br>

# 고급 인덱스 스캔 (EXPLAIN)
## Index Skip Scan
* 발생 조건
    * 선두 컬럼의 distinct 값이 적을 경우
    * 후행 컬럼의 조건 선택도가 높을 경우
    * 테이블 크기가 크고 `full scan`이 부담될 경우
### 크기가 큰 테이블 생성
```sql
set session cte_max_recursion_depth = 500000; -- 재귀의 최대 반복 횟수 설정

create table t1 (c1 int, c2 int, index idx1(c1, c2));

insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select 1, n from seq; -- 500000건의 데이터 삽입
```
### 선두 컬럼의 distinct 값 확인
* 확실하게 `skip scan`을 확인하기 위해 c1에는 1개의 값만 넣음
```sql
select distinct(c1) from t1;
```
```sql
+------+
| c1   |
+------+
|    1 |
+------+
1 row in set (0.00 sec)
```
### 후행 칼럼의 Cardinality 확인
* MySQL 옵티마이저는 컬럼의 `Cardinality`를 기반으로 조건 `Selectivity`를 추정
    * `Selectivity`는 컬럼 단위가 아닌 **조건 단위** 개념이므로 통계 정보로 직접 제공되지 않음
```sql
analyze table t1; -- 통계 정보 갱신
show index from t1;
```
```sql
Table       : t1
Column_name : c1
Cardinality : 1
-----------------------
Table       : t1
Column_name : c2
Cardinality : 499206
```
### 실행 계획 확인
```
explain select * from t1 where c2 = 1;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : range -- index range scan
possible_keys   : idx1
key             : idx1
key_len         : 10 -- c1(4) + c2(4) + null flag(1) = 10 bytes
ref             : NULL
rows            : 49920
filtered        : 100.00
Extra           : Using where; Using index from skip scan -- Index Skip Scan 사용
```
* `type=range`
    * `skip scan`은 내부적으로 `c1`의 각 distict 값에 대해 `c2`에 대해 범위 탐색을 반복함
* `key_len=10`
    * 복합 인덱스의 모든 컬럼 사용
    * 후행 컬럼 조건까지 인덱스 범위에 포함됨
* `rows=49920`
    * `full table scan`의 경우 `499993` row 를 읽음 (추정값)
    * `skip scan`으로 인해 약 10% 수준으로 감소
### +) where절이 범위 조건일 경우 Skip Scan으로 풀릴까?
* `skip scan`의 선택 조건은 동등 조건이냐 범위 조건이냐가 아님
* 범위 조건일 경우, index 탐색 범위가 넓어져 조건 선택도(`selectivity`)가 낮아지므로 `skip scan`이 선택되지 않을 가능성이 높아지는 것
* 결국 판단은 옵티마이저의 몫
## Loose Index Scan
* 그룹마다 필요한 인덱스 엔트리만 골라서 읽고, 나머지는 스킵하는 방식
* 보통 각 그룹의 첫 번째 또는 마지막 엔트리만 읽음
* 발생 조건
    * group by 컬럼이 복합 인덱스의 선두 컬럼이며
    * 후행 컬럼에 집계함수 또는 min()/max() 함수가 있어야 함
    * where 조건이 인덱스의 정렬 구조를 깨면 안됨
        * where 조건이 인덱스 정렬 순서를 무력화하여 그룹별 첫/마지막 인덱스 엔트리가 min/max 값이라는 보장을 깰 경우
* 과거 MySQL 버전에서는 `Extra=Using index for group-by` 표시로 `loose index scan` 사용 여부를 추론할 수 있었음
    * 현재는 `explain`만으로는 `loose index scan` 적용 여부 확인이 어렵거나 불가능함
    * `explain analyze`나 `optimizer trace` 기능을 활용해야 함
### Loose Index Scan으로 풀리는지 확인
* `c1=1~1000`, `c2=1~1000` 의 데이터 삽입
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, index idx1(c1, c2));

insert into t1 with recursive seq(n) as (select 1 union all select n+1 from seq where n < 1000), c1_list(c1) as (select 1 union all select c1+1 from c1_list where c1<1000) select c1, c1*10+n from seq join c1_list;

explain select c1, min(c2) from t1 group by c1; -- type=index
```
* 예상했던 `Loose Index Scan`으로 풀리지 않음
### where 조건을 추가해보자
* 이론적으로 `c1=100`의 첫 row를 읽고 바로 `c1=101`로 점프하는 방식으로 동작한다면 원하는 loose index scan으로 풀릴 것이라 추측
```sql
explain select c1, min(c2) from t1 where c1 between 100 and 200 group by c1; -- type=range
```
```sql
| -> Group aggregate: min(t1.c2)  (cost=60386 rows=200958) (actual time=1.85..53.3 rows=101 loops=1)
    -> Filter: (t1.c1 between 100 and 200)  (cost=40290 rows=200958) (actual time=0.0181..48.2 rows=101000 loops=1)
        -> Covering index range scan on t1 using idx1 over (100 <= c1 <= 200)  (cost=40290 rows=200958) (actual time=0.0169..41.7 rows=101000 loops=1)
 |
```
* range scan 노드에서 실제로 읽은 row 수 = `101000`
* 최종 그룹 수 = `101`
* 만약 `loose index scan`을 했다면 range scan에서 읽은 row 수가 그룹수와 일치하거나 비슷해야 함
* 즉, where 조건에 해당하는 모든 인덱스 엔트리를 읽었고 `loose index scan`으로 풀리지 않음
### DISTINCT를 사용해보자
* `distinct`는 같은 `c1` 값은 한번만 결과에 나오게 하므로 그룹별로 첫 엔트리만 읽고 건너뛰는 loose index scan으로 풀릴 것이라 추측
```sql
explain select distinct c1 from t1; -- type=index
```
* `index full scan`으로 풀림
* 여전히 `Loose Index Scan`으로 풀리지 않음
### 결론
어떤 방식으로 쿼리를 구성하더라도 `loose index scan`으로 풀리는 경우를 확인할 수 없었다. 이는 실제 MySQL 옵티마이저가 이론적으로 `loose index scan`의 적용 조건을 만족하더라도 해당 방식을 쉽게 선택하지 않기 때문으로 보인다.

즉, `loose index scan`은 항상 발생하는 최적화 기법이 아니라, cost 이점이 매우 명확할 때만 선택되는 매우 특수한 접근 방식으로 이해하는 것이 적절하다.
<br>

# Index Merge vs 복합 인덱스
## AND 조건 실행 계획 비교
### Index Merge 실행 계획 확인
* 각각의 컬럼에 단일 인덱스 생성 후 `index merge` 실행 계획 확인
```sql
create table t1(c1 int, c2 int, index idx1(c1), index idx2(c2));
insert into t1 values (1, 1), (1, 2), (1, 3), (2, 1), (2, 2), (2, 3);

explain select * from t1 where c1 = 1 and c2 = 2;
explain analyze select * from t1 where c1 = 1 and c2 = 2;
```
```sql
type            : index_merge -- index merge 사용
possible_keys   : idx1, idx2
key             : idx2, idx1 -- 순서는 중요하지 않음
key_len         : 5, 5 -- 각 인덱스에서 선두 컬럼 사용
Extra           : Using intersect(idx2, idx1); Using where; Using index
```
* where 조건이 `and`로 연결되어 있기 때문에 `index merge(intersect)` 수행
* intersect 내부의 인덱스 순서는 중요하지 않음
    * 어떤 인덱스를 교집합한 것이 중요
```sql
| -> Filter: ((t1.c2 = 2) and (t1.c1 = 1))  (cost=0.601 rows=1) (actual time=0.0421..0.044 rows=1 loops=1)
    -> Intersect rows sorted by row ID  (cost=0.601 rows=1) (actual time=0.0405..0.0424 rows=1 loops=1)
        -> Index range scan on t1 using idx2 over (c2 = 2)  (cost=0.25 rows=2) (actual time=0.0156..0.0161 rows=2 loops=1)
        -> Index range scan on t1 using idx1 over (c1 = 1)  (cost=0.251 rows=3) (actual time=0.0195..0.0242 rows=3 loops=1)
 |
```
* 실행 순서
    * rowid 집합 생성
        * 두개의 range scan은 독립적으로 수행됨
        * `Index range scan on t1 using idx2`
            * idx2를 range scan해서 2개의 rowid 집합 생성
        * `Index range scan on t1 using idx1`
            * idx1을 range scan해서 3개의 rowid 집합 생성
    * `Intersect rows sorted by row ID` (★★)
        * 두 개의 rowid 집합을 rowid 기준으로 정렬 후 교집합
        * 인덱스마다 rowid 순서가 다를 수 있으므로 효율적인 교집합 연산을 위해 정렬 필수
    * `Filter: ((t1.c2 = 2) and (t1.c1 = 1))`
        * index merge 이후 서버 레벨에서 조건 재검증
            * null 처리
            * 표현식(expression) 계산
            * MVCC 기반 row 가시성 확인
            * false positive 제거를 위한 보수적 조건 검증

### 복합 인덱스를 사용한 쿼리 실행 계획 확인
```sql
create index idx3 on t1(c1, c2);

explain select * from t1 where c1 = 1 and c2 = 2;
explain analyze select * from t1 where c1 = 1 and c2 = 2;
```
```sql
type            : ref
possible_keys   : idx1, idx2, idx3
key             : idx3
key_len         : 10
Extra           : Using index
```
```sql
| -> Covering index lookup on t1 using idx3 (c1=1, c2=2)  (cost=0.35 rows=1) (actual time=0.0153..0.0186 rows=1 loops=1)
 |
```
* 복합 인덱스를 사용한 경우 `index merge`를 한 것보다 실행 시간이 단축된 것을 확인할 수 있음
    * index merge의 actual time : `0.0421..0.044`
    * 복합 인덱스의 actual time : `0.0153..0.0186`
## OR 조건 실행 계획 비교
### Index Merge 실행 계획 확인
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int, index idx1(c1), index idx2(c2));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n+1 from seq; -- 500000건의 데이터 삽입

explain select * from t1 where c1 = 1 or c2 = 2;
explain analyze select * from t1 where c1 = 1 or c2 = 2;
```
```sql
type            : index_merge
possible_keys   : idx1, idx2
key             : idx1, idx2 
key_len         : 5, 5 
Extra           : Using union(idx1,idx2); Using where;
```
* where 조건이 `or`로 연결되어 있기 때문에 `index merge(union)` 수행
```sql
| -> Filter: ((t1.c1 = 1) or (t1.c2 = 2))  (cost=1.52 rows=2) (actual time=0.0654..0.0662 rows=1 loops=1)
    -> Deduplicate rows sorted by row ID  (cost=1.52 rows=2) (actual time=0.0638..0.0645 rows=1 loops=1)
        -> Index range scan on t1 using idx1 over (c1 = 1)  (cost=0.36 rows=1) (actual time=0.0392..0.0397 rows=1 loops=1)
        -> Index range scan on t1 using idx2 over (c2 = 2)  (cost=0.36 rows=1) (actual time=0.00581..0.00894 rows=1 loops=1)
 |
```
* `Deduplicate rows sorted by row ID` (★★)
    * 두 개의 rowid 집합을 rowid 기준으로 정렬 후 중복을 제거 한 후 병합
### 복합 인덱스를 사용한 쿼리 실행 계획 확인
```sql
drop index idx1 on t1;
drop index idx2 on t1;
create index idx3 on t1(c1, c2);

explain select * from t1 where c1 = 1 or c2 = 2;
explain analyze select * from t1 where c1 = 1 or c2 = 2;
```
```sql
type            : index -- index full scan
possible_keys   : idx3
key             : idx3
key_len         : 10
Extra           : Using where; Using index;
```
```sql
| -> Filter: ((t1.c1 = 1) or (t1.c2 = 2))  (cost=50201 rows=49922) (actual time=0.0697..225 rows=1 loops=1)
    -> Covering index scan on t1 using idx3  (cost=50201 rows=499206) (actual time=0.068..193 rows=500000 loops=1)
 |
```
* `and 조건`의 경우와는 다르게, 복합 인덱스를 사용한 쿼리의 전체 실행시간이 더 느린 것을 확인할 수 있음
    * index merge의 actual time : `0.0654..0.0662`
    * 복합 인덱스의 actual time : `0.0697..225`
* `or 조건` 특성 상, 복합 인덱스를 사용하더라도 각 조건을 동시에 만족하는 범위 탐색이나 index lookup이 불가능한 경우가 많음
    * 결국 `index full scan` 후 필터링 하는 방식으로 동작
* 즉, **`or 조건` 쿼리에서는 데이터가 증가할수록 복합 인덱스를 사용하는 것보다 `index merge`를 하는 것이 더 유리할 수 있음**

## 결론
`and 조건`이 있는 쿼리의 경우, 일반적으로 `복합 인덱스`가 index merge 보다 성능 상 유리하다. 따라서 자주 조회되는 컬럼 조합이 있다면 index merge 에 의존하기보다는 명시적인 `복합 인덱스`를 설계하는 것이 바람직하다.

`or 조건`이 있는 쿼리의 경우, 데이터 규모와 조건 선택도에 따라 복합 인덱스보다 `index merge`가 더 효율적인 실행 계획이 될 수 있다.