# 목차
* Histogram 유무에 따른 selectivity 추정 변화 확인 실습
    * histogram이 없는 경우
    * histogram이 있는 경우
* 참고 자료
  * [옵티마이저와 통계정보](../optimizer-statistics.md)
  * [explain 해석](../explain.md)
<br><br>

# Histogram 유무에 따른 selectivity 추정 변화 확인 실습
## histogram이 없는 경우
### 실험 준비
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int);
insert into t1 with recursive seq as
(
    select 1 as n
    union all
    select n+1 from seq where n < 500000
)
select
    case
        when n <= 499999 then 1
        else n
    end as c1      -- c1 대부분은 1로 skewed data
from seq;
```
### 실제 분포 확인
```sql
select c1, count(*) from t1 group by c1;
```
```sql
+--------+----------+
| c1     | count(*) |
+--------+----------+
|      1 |   499999 |
| 500000 |        1 |
+--------+----------+
2 rows in set (0.28 sec)
```
### 실행 계획 확인
```sql
explain select * from t1 where c1 = 1;

+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 499550 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
```sql
explain select * from t1 where c1 = 500000;

+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 499550 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
## histogram이 있는 경우
### histogram 생성
```sql
analyze table t1 update histogram on c1 with 32 buckets;
```
### 실행 계획 확인
```sql
explain select * from t1 where c1 = 1;

+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 499550 |   100.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
```sql
explain select * from t1 where c1 = 500000;

+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | t1    | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 499550 |     0.10 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
## 결과
* `c1 = 1`
    * histogram 생성 전
        * rows : `499550`
        * filtered : `10.00`
        * 최종 예상 결과 row ≈ rows * filtered / 100 = 499550 * 10 % = `49955`
    * histogram 생성 후
        * rows :  `499550`
        * filtered : `100.00`
        * 최종 예상 결과 row ≈ 499550 * 100 % = `499550`
* `c1 = 500000`
    * histogram 생성 전
        * rows : `499550`
        * filtered : `10.00`
        * 최종 예상 결과 row ≈ 499550 * 10 % = `49955`
    * histogram 생성 후
        * rows : `499550`
        * filtered : `0.10`
        * 최종 예상 결과 row ≈ 499550 * 0.1 % = `499.55`
## 결론
실험 결과, histogram은 skewed data 환경에서 옵티마이저가 컬럼 분포를 더 정확하게 인식하도록하여 selectivity 추정 정확도를 높이는 역할을 한다는 것을 확인할 수 있다.

histogram이 없는 경우, 옵티마이저는 `c1 = 1`과 `c1 = 500000`을 동일한 selectivity (`filtered = 10%`)로 추정했다. 이는 옵티마이저가 실제 데이터 분포를 알 수 없기 때문에 heuristic 기반 selectivity 추정을 수행한 결과이다. 즉, 실제 데이터 분포를 알지 못한 상태에서는 서로 완전히 다른 조건이라도 비슷한 selectivity로 판단할 수 있음을 의미한다.

반면, histogram 생성 후에는 `c1 = 1`과 `c1 = 500000`을 서로 다른 selectivity로 추정했다. `c1 = 1`의 경우, 실제 분포에 맞는 높은 selectivity(`filtered = 100%`)를 사용하여 실제 조건에 만족하는 row 수와 거의 유사한 추정값을 도출했다. `c1 = 500000`의 경우, 실제 분포에 맞는 낮은 selectivity(`filtered = 0.10%`)를 사용하여 극소수만 조건을 만족하는 값으로 인식했지만, 최종 예상 결과 row는 실제(1 row)와 완전히 일치하지는 않았다. 하지만 histogram 생성 전보다 실제 데이터 분포에 훨씬 가까운 방향으로 selectivity 추정이 개선되었으므로 유의미한 결과를 도출했다고 해석할 수 있다.

+) 추가로, histogram의 주된 목적은 access type 자체를 직접 변경하는 것이 아니다. 따라서 histogram이 존재한다고 해서 반드시 `ref`, `range`, `ALL`과 같은 실행 계획이 변경되는 것은 아니다. 특히, 인덱스가 존재하여 range optimizer가 인덱스를 이용한 access path를 분석하는 경우, 일부 상황에서 histogram보다 index 통계 기반 row estimate가 사용될 수 있다. 즉, histogram이 존재하거나 갱신되었더라도 실행 계획이나 추정 결과가 크게 달라지지 않는 경우가 있을 수 있다.
<br><br>