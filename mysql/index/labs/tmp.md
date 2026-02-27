# 목차
* 정렬 + 인덱스 동작 검증
    * filesort vs 인덱스 정렬 성능 비교
* 정렬 방향에 따른 인덱스 정렬
    * 오름차순(ASC)
    * 내림차순(DESC)
    * 혼합 정렬
* 상황별 인덱스 정렬 검증
    * equality 조건 + order by
    * range 조건 + order by
    * equality + range 조건 + order by
    * left-most rule과 order by
    * order by + limit
    * where 조건 + order by + limit
* 참고 자료
<br>

# 정렬 + 인덱스 동작 검증
## filesort vs 인덱스 정렬 성능 비교
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000) from seq;
```
### filesort
```sql
explain select * from t1 order by c1;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL
key             : NULL
key_len         : NULL
rows            : 500000
filtered        : 100
Extra           : Using filesort -- filesort를 사용하여 정렬
```
### 인덱스 정렬
```sql
create index idx1 on t1(c1);
analyze table t1;

explain select * from t1 order by c1;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL
key             : NULL
key_len         : NULL
rows            : 499206
filtered        : 100
Extra           : Using filesort -- filesort를 사용하여 정렬
```
* Optimizer Trace 확인
    * 인덱스가 존재하지만 filesort를 사용하여 정렬한 원인을 찾기위해 분석
```sql
set optimizer_trace = 'enabled=on';
explain select * from t1 order by c1;
select * from information_schema.optimizer_trace;
```
```sql
-- Oprimizer Trace 결과 중 필요한 부분만 발췌

"best_access_path": { -- 옵티마이저가 선택한 최적의 path
    "considered_access_paths": [
        {
            "rows_to_scan": 499206,
            "access_type": "scan", -- full table scan
            "resulting_rows": 499206,
            "cost": 50201.4,
            "chosen": true, -- 현재 path로 확정
            "use_tmp_table": true
        }
    ]
},

"reconsidering_access_paths_for_index_ordering": { -- 정렬을 인덱스로 처리할 수 있는지 재검토
    "clause": "ORDER BY",
    "steps": [], -- 실제로 시도한 변경 과정 없음
    "index_order_summary": {
        "table": "`t1`",
        "index_provides_order": false, -- 인덱스를 사용해서 정렬을 처리할 수 없음
        "order_direction": "undefined",
        "index": "unknown", -- 정렬에 사용할 인덱스 없음
        "plan_changed": false -- order by를 고려했지만 실행 계획은 바뀌지 않음
    }
}
```
### 결과 비교
* 인덱스가 없는 경우 : 정상적으로 filesort를 사용하여 정렬 수행
* 인덱스가 존재하는 경우
    * 정렬을 위해 인덱스를 사용할수 있는지 검토
    * 인덱스가 정렬을 제공하지 못한다고 판단
    * filesort를 사용해서 정렬 수행
### 결론
where 조건 없이 전체 데이터를 정렬하는 실험을 수행한 결과, 인덱스가 존재 여부에 상관없이 filesort를 사용해서 정렬을 수행했다. 이는 인덱스를 활용한 정렬이 이론적으로 가능하더라도, 많은 row에 대한 random I/O(back lookup) 비용으로 인해 full table scan + filesort 방식이 더 낮은 cost로 평가되어 선택되었기 때문이다.

Optimizer Trace 결과를 분석한 결과, 실제로 `considered_access_paths`에 인덱스를 사용한 scan이 후보로 조차 고려되지 않은 것을 확인할 수 있다. MySQL 옵티마이저는 모든 가능한 access path를 항상 생성하지 않으며, heuristic pruning 과정에서 index ordering 후보가 cost 비교 이전 단계에서 제외되었음을 의미한다.

따라서, filesort와 인덱스 정렬의 성능 차이를 비교하기 위해서는 추가적인 실험 환경이 필요하다.
## filesort vs 인덱스 정렬 성능 비교 - where 조건
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, c3 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;
```
### filesort
```sql
explain select * from t1 where c1 = 10 order by c2;
explain analyze select * from t1 where c1 = 10 order by c2;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL
key             : NULL
key_len         : NULL
rows            : 499365
filtered        : 10.00
Extra           : Using where; Using filesort
```
```sql
| -> Sort: t1.c2  (cost=50249 rows=499365) (actual time=217..217 rows=523 loops=1)
    -> Filter: (t1.c1 = 10)  (cost=50249 rows=499365) (actual time=2.07..217 rows=523 loops=1)
        -> Table scan on t1  (cost=50249 rows=499365) (actual time=0.0284..199 rows=500000 loops=1)
 |
 ```
### 인덱스 정렬
```sql
create index idx1 on t1(c1, c2);
analyze table t1;

explain select * from t1 where c1 = 10 order by c2;
explain analyze select * from t1 where c1 = 10 order by c2;
```
```sql
type            : ref
possible_keys   : idx1
key             : idx1
key_len         : 5
rows            : 523
filtered        : 100
Extra           : NULL
```
```sql
| -> Index lookup on t1 using idx1 (c1=10)  (cost=183 rows=523) (actual time=0.0253..1.76 rows=523 loops=1)
 |
```
### 결과 비교
* 인덱스가 없는 경우
    * filesort를 사용하여 정렬 수행
    * 실행 시간 : `217..217`
* 인덱스가 존재하는 경우
    * 인덱스를 사용하여 정렬 수행
    * 실행 시간 : `0.0253..1.76`
### 결론
where 조건을 추가하여 실험해 본 결과, 인덱스가 없는 경우에는 full table scan + filesort를 사용하여 정렬을 수행하였고, 복합 인덱스가 존재하는 경우에는 인덱스를 활용한 정렬이 수행되었다.

특히 selectivity가 높은 equality 조건을 사용한 경우, 인덱스 정렬 방식은 조건에 해당하는 row만 탐색한 후 이미 정렬된 B+Tree를 순차적으로 읽음으로써 별도의 정렬 과정을 수행하지 않았다. 그 결과 filesort 방식 대비 실행 시간이 크게 단축되는 것을 확인할 수 있다.

단, 인덱스 정렬이 항상 filesort 방식보다 빠른 것은 아니다. 조회 범위가 넓거나 selectivity가 낮은 조건을 사용할 경우에는 인덱스 random I/O 비용이 커져 filesort 방식이 더 효율적인 경우가 있을 수 있다.
<br>

# 정렬 방향에 따른 인덱스 정렬
## 오름차순(ASC), 내림차순(DESC) 정렬
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, index idx1(c1));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000) from seq;
analyze table t1;
```
### 오름차순(ASC) 정렬
* 위의 실습에서 인덱스를 사용하여 오름차순(acs) 정렬이 가능함을 확인했으므로 생략
### 내림차순(ASC) 정렬
```sql
explain select * from t1 order by c1 desc;
```
```sql
type            : index
possible_keys   : NULL
key             : idx1
key_len         : 5
rows            : 499550
filtered        : 100
Extra           : Backward index scan; Using index
```
### 결과
* 오름차순(asc), 내림차순(desc) 정렬 모두 인덱스를 사용하여 정렬 수행
### 결론
단일 컬럼에 대해 생성된 B+Tree 인덱스를 기준으로 실험해 본 결과, 오름차순과 내림차순 정렬 모두 인덱스를 활용한 정렬이 가능함을 확인할 수 있다.

MySQL InnoDB의 인덱스는 leaf node가 정렬된 상태로 이중 연결 리스트 형태로 연결된 B+Tree 구조를 가진다. 따라서 오름차순 정렬은 leaf node를 순방향으로 탐색하고, 내림차순 정렬은 역방향으로 탐색하는 방식으로 filesort 과정 없이 정렬을 처리할 수 있다.

즉, 인덱스는 정방향 뿐만 아니라 역방향 정렬을 효율적으로 지원한다. 다만, 복합 인덱스의 경우에는 left-most rule과 정렬 방향이 일치해야 인덱스 정렬이 가능하므로 인덱스 설계 시 이를 함께 고려하는 것이 중요하다.
## 혼합 정렬
### table 생성 및 정렬 수행
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, index idx1(c1, c2));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000) from seq;
analyze table t1;

explain select * from t1 order by c1 asc, c2 desc;
explain analyze select * from t1 order by c1 asc, c2 desc;
```
```sql
type            : index -- index full scan
possible_keys   : NULL
key             : idx1
key_len         : 10
rows            : 499206
filtered        : 100
Extra           : Using index; Using filesort -- filesort를 사용하여 정렬
```
### 결과
* 서로 다른 정렬을 혼합한 경우 인덱스를 통해 데이터를 읽었지만 filesort를 사용하여 정렬을 수행
### 결론
혼합 정렬의 경우, 복합 인덱스가 존재함에도 불구하고 인덱스만으로 정렬을 처리하지 못하고 filesort를 사용하여 추가 정렬을 수행했다.

B+Tree 기반 인덱스는 특정 방향으로 정렬된 구조를 가진다. 전체 컬럼에 대해 동일한 방향으로 정렬하는 경우에는 leaf node를 순방향 또는 역방향으로 탐색하여 인덱스 정렬이 가능하다. 그러나 컬럼별 정렬 방향이 서로 다른 경우에는 단순한 순방향 또는 역방향 탐색만으로는 정렬 조건을 만족할 수 없다.
<br>

# 상황별 인덱스 정렬 검증
## range 조건 + order by
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, index idx1(c1, c2));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000) from seq;
analyze table t1;
```
### range 조건 + order by (인덱스 정렬이 깨질 경우)
```sql
explain select * from t1 where c1 > 10 order by c2;
```
```sql
type            : range
possible_keys   : idx1
key             : idx1
key_len         : 5
rows            : 249603
filtered        : 100
Extra           : Using where; Using index; Using filesort;
```
### range 조건 + order by (인덱스 정렬이 유지될 경우)
```sql
set session optimizer_switch='skip_scan=off';

explain select * from t1 where c2 > 20 order by c1;
```
```sql
type            : index
possible_keys   : NULL
key             : idx1
key_len         : 10
rows            : 499206
filtered        : 33.33
Extra           : Using where; Using index;
```
### 결과
* range 조건 + order by
    * 인덱스 정렬이 깨진 경우 : filesort 방식으로 정렬
    * 인덱스 정렬이 유지된 경우 : 인덱스 정렬
### 결론
range 조건이 적용된 컬럼 이후의 컬럼으로 정렬을 수행할 경우, 인덱스 정렬이 깨지고 filesort 방식이 사용되는 것을 확인할 수 있다.

복합 인덱스(c1, c2)는 먼저 c1 값이 정렬되고, 동일한 c1 값 내에서 c2가 정렬된 구조를 가진다. 선두 컬럼에 range 조건이 존재하는 경우(`where c1 > 10 order by c2`), 조건을 만족하는 레코드들은 서로 다른 c1 값을 기준으로 여러 개의 그룹으로 나뉘게 된다. 각 그룹 내부에서는 c2가 정렬되어 있지만, 전체 결과 집합에 대한 c2의 정렬 순서는 보장되지 않는다. 따라서 인덱스를 사용한 정렬이 불가능하다고 판단하고 추가적인 filesort를 수행한다. 

range 조건이 선두 컬럼이 아닌 경우(`where c2 > 20 order by c1`), 인덱스를 순차적으로 탐색하면서 결과를 필터링 할 수 있으며 인덱스의 기본 정렬 순서인 c1이 유지된다. 따라서 추가적인 정렬 과정(filesort) 없이 인덱스를 활용한 정렬이 가능하다.

즉, 복합 인덱스에서 range 조건 컬럼의 위치에 따라 인덱스 정렬 사용 가능 여부가 달라진다.






???? equality + range 조건 + order by가 필요한가?
## equality + range 조건 + order by (인덱스 정렬이 깨질 경우)
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, c3 int, index idx1(c1, c2, c3));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;
analyze table t1;
```
### equality + range 조건 + order by (인덱스 정렬이 깨질 경우)
```sql
explain select * from t1 where c1 = 10 and c2 > 20 order by c3;
explain analyze select * from t1 where c1 = 10 and c2 > 20 order by c3;
```
```sql
type            : range
possible_keys   : idx1
key             : idx1
key_len         : 10
rows            : 453
filtered        : 100
Extra           : Using where; Using index; Using filesort
```
```sql
| -> Sort: t1.c3  (cost=91.1 rows=453) (actual time=0.354..0.368 rows=453 loops=1)
    -> Filter: ((t1.c1 = 10) and (t1.c2 > 20))  (cost=91.1 rows=453) (actual time=0.0173..0.27 rows=453 loops=1)
        -> Index range scan on t1 using idx1 over (c1 = 10 AND 20 < c2)  (cost=91.1 rows=453) (actual time=0.0159..0.228 rows=453 loops=1)
 |
```
### equality + range 조건 + order by (인덱스 정렬이 유지될 경우)
```sql
explain select * from t1 where c1 = 10 and c3 > 30 order by c2;
explain analyze select * from t1 where c1 = 10 and c3 > 30 order by c2;
```
```sql
type            : ref
possible_keys   : idx1
key             : idx1
key_len         : 5
rows            : 462
filtered        : 33.33
Extra           : Using where; Using index;
```
```sql
| -> Filter: (t1.c3 > 30)  (cost=15.9 rows=154) (actual time=0.0787..0.3 rows=454 loops=1)
    -> Covering index lookup on t1 using idx1 (c1=10)  (cost=15.9 rows=462) (actual time=0.043..0.239 rows=462 loops=1)
 |
```
### 결과
* range 조건 + order by : filesort를 사용해서 정렬
* equality + range 조건 + order by
    * 인덱스 정렬이 깨질 경우 : filesort를 사용해서 정렬
    * 인덱스 정렬이 유지될 경우 : 인덱스 정렬
### 결론
## left-most rule과 order by
## order by + limit
## where 조건 + order by + limit














# ORDER BY + LIMIT
* [→ limit에 대한 자세한 내용 보기](../optimization/limit.md)
### limit을 사용하면 쿼리 수행 시간이 단축될까?
#### order by 단독 사용 (limit 없음)
* `idx1`이 있지만 c2를 위해 542241 번의 back lookup이 발생할 것으로 추정
* 인덱스를 사용하는 것보다 테이블 전체 스캔 후 filesort가 더 싸다고 판단
```sql
create table t1 (c1 int, c2 int, index idx1(c1));
insert into t1 select floor(rand()*100000), floor(rand()*100000) from information_schema.columns a cross join information_schema.columns b;
select count(*) from t1; -- 543169 건 삽입

explain select * from t1 order by c1;
explain analyze select * from t1 order by c1;
```
```sql
type = ALL
key = NULL
rows = 542241
Extra = Using filesort
```
```sql
| -> Sort: t1.c1  (cost=54537 rows=542241) (actual time=396..413 rows=543169 loops=1)
    -> Table scan on t1  (cost=54537 rows=542241) (actual time=0.0237..203 rows=543169 loops=1)
 |
```
#### order by + limit
* 인덱스를 사용하여 정렬
* `type=index`이지만 index full scan이 아닌 10건의 row만 읽고 조기 종료
* `limit` 자체가 빠른것이 아니라, 전체 정렬을 하지 않아도 되는 상황이 발생한 것
```sql
explain select * from t1 order by c1 limit 10;
explain analyze select * from t1 order by c1 limit 10;
```
```sql
type = index
key = idx1
rows = 10
Extra = NULL
```
```sql
| -> Limit: 10 row(s)  (cost=0.00579 rows=10) (actual time=0.0781..0.145 rows=10 loops=1)
    -> Index scan on t1 using idx1  (cost=0.00579 rows=10) (actual time=0.0773..0.143 rows=10 loops=1)
 |
```
* order by를 단독 사용한 경우와 비교했을 때 `actual time`이 현저하게 감소한 것을 볼 수 있음
    * order by 단독 사용 actual time : `396..413`
    * order by + limit actual time : `0.0781..0.145`
### where 범위 조건이 있으면 어떻게 동작할까?
#### 정렬 + where(range)
* index range scan한 결과 자체가 이미 c1에 대해 정렬된 상태
* 추가적인 정렬 필요 없음
```sql
explain select * from t1 where c1 < 10 order by c1;
explain analyze select * from t1 where c1 < 10 order by c1;
```
```sql
type = range
key = idx1
rows = 58
Extra = Using index condition -- ICP : 인덱스 레벨에서 조건 필터링
```
```sql
| -> Index range scan on t1 using idx1 over (NULL < c1 < 10), with index condition: (t1.c1 < 10)  (cost=26.4 rows=58) (actual time=0.0581..0.353 rows=58 loops=1)
 |
 ```
#### 정렬 + where(range) + limit
* index range scan을 시작하여 필요한 row 10건을 읽으면 조기 종료
```sql
explain select * from t1 where c1 < 10 order by c1 limit 10;
explain analyze select * from t1 where c1 < 10 order by c1 limit 10;
```
```sql
type = range
key = idx1
rows = 58
Extra = Using index condition
```
```sql
| -> Limit: 10 row(s)  (cost=26.4 rows=10) (actual time=0.107..0.148 rows=10 loops=1)
    -> Index range scan on t1 using idx1 over (NULL < c1 < 10), with index condition: (t1.c1 < 10)  (cost=26.4 rows=58) (actual time=0.106..0.146 rows=10 loops=1)
 |
```
* `limit`을 함께 사용한 경우 `actual time`의 종료시간이 더 빠름
    * limit 미사용 actual time : `0.0581..0.353`
    * limit 사용 actual time : `0.107..0.148`
* limit이 있는 케이스가 첫 row는 늦게 나옴
    * limit이 항상 더 빠른 첫 응답을 보장하는 것은 아님