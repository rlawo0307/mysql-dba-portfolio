# 목차
* left-most rule 검증
    * (a, b) vs (b, a) 실행 계획 비교
    * 선두 컬럼이 조건에 없는 경우
    * 선두 컬럼 및 선두 prefix만 조건에 사용하는 경우
    * 범위 조건 이후 컬럼 무시
* 참고 자료
    * [→ explain에 대한 자세한 내용 보기](../../optimization/explain.md)
    * [→ index 종류에 대한 자세한 내용 보기](../index-basics.md)
    * [→ 각 scan 방식에 대한 개념 자세히 보기](../../index/index-basics.md)
<br>

# left-most rule 검증
## (a, b) vs (b, a) 실행 계획 비교
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, c3 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;
```
### (a, b) 인덱스
```sql
create index idx1 on t1(c1, c2);
analyze table t1;

explain select * from t1 where c1 = 10 and c2 = 20;
explain analyze select * from t1 where c1 = 10 and c2 = 20;

drop index idx1 on t1;
```
```sql
type            : ref
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 10
rows            : 1
filtered        : 100
Extra           : NULL
```
```sql
| -> Index lookup on t1 using idx1 (c1=10, c2=20)  (cost=0.35 rows=1) (actual time=0.0145..0.0145 rows=0 loops=1)
 |
```
### (b, a) 인덱스
```sql
create index idx2 on t1(c2, c1);
analyze table t1;

explain select * from t1 where c1 = 10 and c2 = 20;
explain analyze select * from t1 where c1 = 10 and c2 = 20;
```
```sql
type            : ref
possible_keys   : idx2
key             : idx2 -- 인덱스 사용
key_len         : 10
rows            : 1
filtered        : 100
Extra           : NULL
```
```sql
| -> Index lookup on t1 using idx2 (c2=20, c1=10)  (cost=0.35 rows=1) (actual time=0.0132..0.0132 rows=0 loops=1)
 |
```
### 결과 비교
* (a, b) 인덱스
    * 비교한 조건 : `c1 = 10, c2 = 20`
    * 결과 : index lookup 수행
* (b, a) 인덱스
    * 비교한 조건 : `c2 = 20, c1 = 10`
    * 결과 : index lookup 수행
        * 인덱스 순서와 쿼리 조건이 다르지만 인덱스를 사용함
### 결론
(a, b) 인덱스의 경우, 인덱스 순서와 쿼리 조건이 left-most rule과 완전히 일치하므로 예상대로 index lookup이 수행되었다.

(b, a) 인덱스의 경우, 쿼리 조건과 실제 비교한 조건의 순서가 달라서 인덱스를 사용하지 않을 것으로 예상했으나, 실제로는 인덱스를 정상적으로 사용한 것을 확인할 수 있다. 이는 SQL은 선언형 언어로, 옵티마이저는 내부적으로 쿼리 조건 순서와 무관하게 조건을 재정렬하기 때문이다. 

즉, 인덱스 순서와 where 절에 작성된 조건의 순서는 중요하지 않으며 left-most 컬럼에 대한 조건이 존재하고 이어지는 컬럼 조건이 있다면 복합 인덱스를 정상적으로 사용할 수 있다.
## 선두 컬럼이 조건에 없는 경우
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, c3 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;

create index idx1 on t1(c1, c2);
analyze table t1;

explain select * from t1 where c2 = 20;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL -- idx1을 사용 가능한 인덱스 후보로 조차 생각하지 않음
key             : NULL
key_len         : NULL
rows            : 499365
filtered        : 10.00
Extra           : Using where
```
### 결과
* 선두 컬럼 조건이 없는 경우 : 인덱스를 사용하지 않음
### 결론
조건에 인덱스의 선두 컬럼이 사용되지 않을 경우, 옵티마이저는 인덱스를 사용하지 않고 full table scan을 선택했다. 옵티마이저는 인덱스를 효율적인 access path로 판단하지 않았기 때문에 possible_keys에도 포함하지 않았다.

(c1, c2) 인덱스는 내부적으로 c1을 기준으로 정렬된 후 같은 c1 값 내에서 c2가 정렬되는 구조를 가진다. 하지만 조건에서는 c1을 사용하지 않고 있다. 즉, c1 값이 무엇인지 모르는 상태에서 c2=20인 row를 찾기 위해서는 전체 인덱스를 전부 탐색해야 한다. 이런 상황에서 옵티마이저는 인덱스를 사용하는 것보다 full table scan의 cost가 더 낮다고 판단했다.

즉, 복합 인덱스는 선두 컬럼을 기준으로 탐색 범위를 좁히는 구조이므로, 선두 컬럼 조건이 없으면 효율적인 탐색이 어려워 옵티마이저가 복합 인덱스를 사용하지 않을 수 있다.
## 선두 컬럼 및 선두 prefix만 조건에 사용하는 경우
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, c3 int, c4 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;

create index idx1 on t1(c1, c2, c3);
analyze table t1;
```
```sql
explain select * from t1 where c1 = 10; -- 선두 컬럼만 사용
```
```sql
type            : ref
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5 -- c1(4) + null flag(1)
rows            : 492
filtered        : 100.00
Extra           : NULL
```
```sql
explain select * from t1 where c1 = 10 and c2 = 20; -- 선두 prefix(c1, c2) 사용
```
```sql
type            : ref
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 10 -- (c1 + null flag) + (c2 + null flag)
rows            : 2
filtered        : 100.00
Extra           : NULL
```
### 결과
* 선두 컬럼 조건 : 인덱스 사용
* 선두 prefix 조건 : 인덱스 사용
### 결론
복합 인덱스의 선두 컬럼부터 연속된 prefix 범위까지 조건이 존재하는 경우 인덱스를 활용할 수 있다. 실행 계획에서도 이를 확인할 수 있으며, key_len이 5에서 10으로 증가한 것은 인덱스 탐색에 사용된 컬럼 범위가 확장되었음을 의미한다. 이는 조건이 많아질수록 탐색 범위가 더욱 좁아질 수 있음을 보여준다.
## 범위 조건 이후 컬럼 무시
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, c3 int, c4 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select floor(rand()*1000), floor(rand()*1000), floor(rand()*1000), floor(rand()*1000) from seq;

create index idx1 on t1(c1, c2, c3);
analyze table t1;

explain select * from t1 where c1 = 10 and c2 < 20 and c3 = 30;
explain analyze select * from t1 where c1 = 10 and c2 < 20 and c3 = 30;
```
```sql
type            : range
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 10 -- c1, c2 컬럼만 사용
rows            : 11
filtered        : 10.00
Extra           : Using index condition -- ICP 방식으로 조건 필터링
```
```sql
| -> Index range scan on t1 using idx1 over (c1 = 10 AND NULL < c2 < 20), with index condition: ((t1.c3 = 30) and (t1.c1 = 10) and (t1.c2 < 20))  (cost=5.21 rows=11) (actual time=0.0454..0.0454 rows=0 loops=1)
 |
```
### 결과
* 인덱스는 사용됐으나 일부 컬럼(c1, c2)만 사용됨
* 범위 조건 이후의 컬럼(c3)은 인덱스 탐색에 사용되지 않음
### 결론
복합 인덱스에서 범위 조건이 등장하면 그 이후 컬럼은 인덱스 탐색 범위를 줄이는 조건으로 사용되지 않는다.

(c1, c2, c3) 인덱스는 내부적으로 c1 → c2 → c3 순으로 정렬된 B+Tree 구조를 가진다. `c1=10` 조건은 탐색 범위를 하나로 좁힐 수 있으므로 계속 인덱스를 따라 내려갈 수 있다. 하지만 `c2 < 20`과 같은 range 조건이 등장하면 탐색 범위가 하나의 구간이 아닌 여러 범위로 확장된다. 이 시점 이후에는 인덱스가 이후 컬럼을 사용하여 탐색 범위를 줄일 수 없게 된다. 따라서 (c1, c2)까지만 인덱스 탐색에 사용되고 `c3=30` 조건은 Index Condition Pushdown(ICP) 방식으로 인덱스 레코드를 읽는 과정에서 추가 필터링을 수행한다.