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