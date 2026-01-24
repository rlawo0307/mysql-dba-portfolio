# 목차
* 고급 인덱스 스캔 (EXPLAIN)
    * Index Skip Scan
    * Loose Index Scan
* Index Merge vs 복합 인덱스 실행 계획 비교
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