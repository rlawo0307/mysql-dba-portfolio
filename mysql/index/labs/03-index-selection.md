# 목차
* 인덱스 선택과 옵티마이저의 비용 기반 판단
    * selectivity에 따른 index 사용 여부 분석
        * 높은 selectivity
        * 낮은 selectivity
    * Disk I/O 방식에 따른 성능 비교
        * Random I/O
        * Sequential I/O
* 참고자료
    * [→ explain에 대한 자세한 내용 보기](../../optimization/explain.md)
    * [→ 각 scan 방식에 대한 개념 자세히 보기](../../index/index-basics.md)
    * [→ selectivity 및 통계정보에 대한 개념 자세히 보기](../../optimization/optimizer-statistics.md)
<br>

# 인덱스 선택과 옵티마이저의 비용 기반 판단
## selectivity에 따른 index 사용 여부 분석
* selectivity
    * 조건을 통과하는 row 비율
    * 결과 row / 전체 row
    * 값이 작을수록 selectivity가 높다고 표현
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, index idx(c1));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n from seq;
```
### selectivity가 높은 조건 조회
* `where c1 = 10` 은 많은 row를 거를 수 있으므로 selectivity가 높음
    * 전체 row = 500000
    * 예상 결과 row = 1
    * selectivity = `1 / 500000`
```sql
explain select * from t1 where c1 = 10;
explain analyze select * from t1 where c1 = 10;
```
```sql
type            : ref -- Non Unique Index + `=` 조건
possible_keys   : idx 
key             : idx -- 인덱스 사용
key_len         : 5
rows            : 1 -- 1개의 결과 row 추출할 것이라 예상
filtered        : 100.00 -- where 조건을 100% 통과할 것이라 예상
Extra           : NULL
```
```sql
| -> Index lookup on t1 using idx (c1=10)  (cost=0.35 rows=1) (actual time=0.0261..0.0286 rows=1 loops=1)
 |
```
### selectivity가 낮은 조건 조회
* `where c1 != 10` 은 대부분의 row를 거를 수 없으므로 selectivity가 낮음
    * 전체 row = 500000
    * 예상 결과 row = 499206
    * selectivity = `499206 / 500000`
```sql
explain select * from t1 where c1 != 10;
explain analyze select * from t1 where c1 != 10;
```
```sql
type            : ALL -- full table scan
possible_keys   : idx 
key             : NULL -- 인덱스는 있지만 사용하지 않음
key_len         : NULL
rows            : 499206
filtered        : 50.00
Extra           : Using where
```
```sql
| -> Filter: (t1.c1 <> 10)  (cost=50201 rows=249612) (actual time=0.0142..210 rows=499999 loops=1)
    -> Table scan on t1  (cost=50201 rows=499206) (actual time=0.0129..183 rows=500000 loops=1)
 |
```
### 결과 비교
* selectivity가 높은 조건 조회
    * 조건 : `where c1 = 10`
    * selectivity : `1 / 500000`
    * index lookup 선택 (`Random I/O`)
    * 실행 시간 : `0.0261..0.0286`
* selectivity가 낮은 조건 조회
    * 조건 : `where c1 != 10`
    * selectivity = `499206 / 500000`
    * table full scan 선택 (`Sequential I/O`)
    * 실행 시간 : `0.0142..210`
### 결론
selectivity가 높은 조건에서는 인덱스를 사용한 index lookup이 선택되었고 매우 빠른 실행 속도를 보였다. 이는 조건이 전체 데이터 중 극히 일부만 조회하므로, Random I/O의 비용이 크지 않기 때문이다. 반면, selectivity가 낮은 조건에서는 인덱스가 존재함에도 불구하고 table full scan을 선택했다. 이 경우 대부분의 row를 읽어야 하므로 인덱스를 사용할 경우 많은 Random I/O를 발생하여 성능이 저하될 수 있기 때문이다.

즉, 옵티마이저는 단순히 인덱스 존재 여부가 아닌 selectivity를 기반으로 비용을 계산한다.

단, 현재 테스트에서는 Random I/O의 경우, 조건(`c1=10`)을 만족하는 row가 극히 일부이기 때문에 Sequential I/O보다 빠른 실행 결과가 나타났다. 그러나 이는 특정 조건에서의 결과 일 뿐, 항상 Random I/O가 Sequential I/O보다 빠른 것은 아니다. 경우에 따라 Sequential I/O가 Random I/O보다 훨씬 효율적일 수 있다. 이를 확인하기 위해서는 추가적인 실험이 필요하다.
## Disk I/O 방식에 따른 성능 비교
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, index idx(c1));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select 1, n from seq;
```
### Random I/O
```sql
explain select * from t1 where c1 = 1;
explain analyze select * from t1 where c1 = 1;
```
```sql
type            : ref -- Non Unique Index + `=` 조건
possible_keys   : idx 
key             : idx -- 인덱스 사용
key_len         : 5
rows            : 249603 
filtered        : 100.00
Extra           : NULL
```
```sql
| -> Index lookup on t1 using idx (c1=1)  (cost=25803 rows=249603) (actual time=0.0169..513 rows=500000 loops=1)
 |
```
### Sequential I/O
```sql
explain select * from t1 ignore index (idx) where c1 = 1;
explain analyze select * from t1 ignore index (idx) where c1 = 1;
-- ignore index를 사용하여 의도적으로 인덱스 무시
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL 
key             : NULL
key_len         : NULL
rows            : 499206
filtered        : 100.00
Extra           : Using where
```
```sql
| -> Filter: (t1.c1 = 1)  (cost=50201 rows=499206) (actual time=0.0253..216 rows=500000 loops=1)
    -> Table scan on t1  (cost=50201 rows=499206) (actual time=0.0238..188 rows=500000 loops=1)
 |
```
### 실행 시간 비교
* Random I/O : `0.0169..513`
* Sequential I/O : `0.0253..216`
### 결론
같은 테이블에서 disk I/O 방식을 다르게 해서 실험해 본 결과, Sequential I/O가 Random I/O보다 약 2배가량 빠르게 나타났다. 이는 반환되는 row수가 많은 낮은 selectivity 조건에서는 인덱스를 사용하는 것이 항상 성능에 유리하지 않음을 보여준다.

단, 현재 실험은 옵티마이저의 실제 선택 결과가 아닌 성능 비교를 위한 의도적인 실험이며, 실제 환경에서는 옵티마이저가 자동으로 full table scan을 선택하는 경우도 존재한다.