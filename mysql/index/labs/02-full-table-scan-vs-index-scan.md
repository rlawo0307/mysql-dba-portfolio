# 목차
* Full Table Scan vs Index Full Scan
* Full Table Scan vs Index Lookup
* 참고자료
    * [→ explain에 대한 자세한 내용 보기](../optimization/explain.md)
    * [→ 각 scan 방식에 대한 개념 자세히 보기](../index/index-basics.md) 
<br>

# Full Table Scan vs Index Full Scan
## 1. 전체 데이터를 조회할 경우 (non covering index)
### full table scan
```sql
create table t1(c1 int, c2 int);
explain select * from t1;
explain analyze select * from t1;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL 
key             : NULL -- 인덱스 미사용
key_len         : NULL
Extra           : NULL
```
```sql
| -> Table scan on t1  (cost=0.35 rows=1) (actual time=0.018..0.018 rows=0 loops=1)
 |
```
### index full scan
```sql
create table t2(c1 int, c2 int, index idx(c1));
explain select * from t2; -- index full scan이 선택될 가능성이 있음
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL
key             : NULL
key_len         : NULL
Extra           : NULL
```
### 결론
인덱스가 존재하더라도 index full scan이 아닌 full table scan으로 풀리는 것을 확인할 수 있다. 이는 옵티마이저가 내부적으로 인덱스가 존재하더라도 full table scan이 cost가 더 낮다고 판단했음을 보여준다.

index full scan은 secondary index를 전체 스캔한 후 각 row마다 clustered index를 조회해야 하므로 랜덤 I/O가 증가할 수 있다. 반면 full table scan의 경우, InnoDB에서는 테이블 데이터가 clustered index 구조로 저장되므로 실제로는 clustered index를 순차적으로 스캔(sequential I/O)하는 것과 동일하다. 따라서 추가적인 lookup 비용이 발생하지 않기 때문에 전체 조회에서는 더 효율적인 경우가 많다.
## 2. non covering index + 충분한 데이터가 존재하는 경우
### full table scan
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10 from seq;
explain select * from t1;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL 
key             : NULL -- 인덱스 미사용
key_len         : NULL
Extra           : NULL
 ```
### index full scan
```sql
set session cte_max_recursion_depth = 500000;

create table t2 (c1 int, c2 int, index idx(c1));
insert into t2 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10 from seq;
explain select * from t2;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL 
key             : NULL -- 인덱스 미사용
key_len         : NULL
Extra           : NULL
```
### 결론
충분한 데이터를 삽입한 후 테스트 해본 결과, 여전히 인덱스가 존재함에도 불구하고 full table scan으로 풀리는 것을 확인할 수 있다.

covering index가 아닌 경우, secondary index scan 이후 clustered index lookup이 추가적으로 발생한다. 이 과정에서 랜덤 I/O 비용이 증가하기 때문에 대량 데이터 조회에서는 full table scan이 더 유리한 경우가 많다.
## 3. covering index를 사용할 경우
### full table scan
```sql
create table t1(c1 int, c2 int);
explain select c1 from t1;
explain analyze select c1 from t1;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL 
key             : NULL -- 인덱스 미사용
key_len         : NULL
Extra           : NULL
```
```sql
| -> Table scan on t1  (cost=0.35 rows=1) (actual time=0.0194..0.0194 rows=0 loops=1)
 |
```
### index full scan
```sql
create table t2(c1 int, c2 int, index idx(c1));
explain select c1 from t2;
explain analyze select c1 from t2;
```
```sql
type            : index -- index full scan
possible_keys   : NULL -- where 조건이 없어서 후보 인덱스를 고려할 필요 없음
key             : idx -- 인덱스 사용
key_len         : 5
Extra           : Using index -- covering index 사용
```
```sql
| -> Covering index scan on t2 using idx  (cost=0.35 rows=1) (actual time=0.0216..0.0216 rows=0 loops=1)
 |
```
### 실행 시간 비교
* Full Table Scan : `0.0194..0.0194`
* Index Full Scan : `0.0216..0.0216`
### 결론
covering index를 사용할 경우, index full scan으로 풀리는 것을 확인할 수 있다. 이는 필요한 컬럼이 모두 인덱스에 포함되어 있어 추가적인 테이블 접근이 필요하지 않기 때문이다.

하지만 index full scan이 full table scan보다 빠를 것이라는 예상과는 다르게 실행 시간 차이는 거의 발생하지 않았다. 이는 데이터가 존재하지 않는 상태에서는 실제 I/O가 발생하지 않기 때문이다. 측정된 `actual time`의 차이는 데이터 접근 비용이 아닌 실행 오버헤드에 의한 미세한 차이일 가능성이 높다.

정확한 성능 비교를 위해서는 충분한 데이터가 존재하는 환경에서 디스크 I/O를 고려한 추가 실험이 필요하다.
## 4. covering index + 충분한 데이터가 존재하는 경우
### full table scan
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10 from seq;
explain analyze select c1 from t1;
```
```sql
| -> Table scan on t1  (cost=50000 rows=500000) (actual time=0.0283..168 rows=500000 loops=1)
 |
```
### index full scan
```sql
set session cte_max_recursion_depth = 500000;

create table t2(c1 int, c2 int, index idx(c1));
insert into t2 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10 from seq;
explain analyze select c1 from t2;
```
```sql
| -> Covering index scan on t2 using idx  (cost=50000 rows=500001) (actual time=0.0273..167 rows=500000 loops=1)
 |
```
### 실행 시간 비교
* Full Table Scan : `0.0283..168`
* Index Full Scan : `0.0273..167`
### 결론
충분한 데이터를 삽입한 환경에서도 full table scan과 index full scan의 실행 시간 차이가 크지 않은 것을 확인할 수 있다. 

InnoDB에서는 테이블 데이터가 clustered index 형태로 저장되기 때문에 full table scan은 실제로 clustered index full scan과 동일하다. covering index 역시 추가적인 테이블 lookup이 발생하지 않기 때문에 둘의 실행시간 차이가 크지 않은 것으로 해석할 수 있다.

또한, row 크기가 작아 secondary index와 clustered index의 크기 차이가 크지 않기 때문에 두 방식의 I/O 비용 차이가 크게 발생하지 않았다. 일반적으로 컬럼 수가 많거나 row 크기가 클수록 covering index의 성능 이점이 더 크게 나타날 수 있다.
## 5. covering index + 충분한 데이터 +  big table의 경우
### full table scan
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int, c3 char(255));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10, repeat('x', 255) from seq;
explain analyze select c1 from t1;
```
```sql
| -> Table scan on t1  (cost=50000 rows=500000) (actual time=0.0755..300 rows=500000 loops=1)
 |
```
### index full scan
```sql
set session cte_max_recursion_depth = 500000;

create table t2(c1 int, c2 int, c3 char(255), index idx(c1));
insert into t2 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10, repeat('x', 255)from seq;
explain analyze select c1 from t2;
```
```sql
| -> Covering index scan on t2 using idx  (cost=50001 rows=500000) (actual time=0.0916..189 rows=500000 loops=1)
 |
```
### 실행 시간 비교
* Full Table Scan : `0.0755..300`
* Index Full Scan : `0.0916..189`
### 결론
size가 큰 컬럼을 추가하여 테스트 해본 결과, 두 scan 방식의 실행 시간 차이가 뚜렷해진 것을 확인할 수 있다.

covering index의 경우, secondary index leaf page에 필요한 컬럼만 저장되어 있기 때문에 상대적으로 작은 데이터만 읽으면 된다. 반면 full table scan의 경우, clustered index 전체 row를 읽어야 하므로 row 크기가 커질수록 더 많은 page I/O가 발생한다.

따라서, 데이터 크기가 커질수록 covering index I/O 절감 효과가 크게 나타나며, 그 결과 index full scan이 full table scan보다 더 빠르게 수행될 수 있다.
<br>

# Full Table Scan vs Index Lookup
* 실제 back lookup이 발생하는 경우와 full table scan의 성능 차이를 비교해보자
## 1. 전체 데이터를 조회할 경우 (non covering index)
### full table scan
```sql
create table t1(c1 int, c2 int);
explain select * from t1 where c1 = 10;
explain analyze select * from t1 where c1 = 10;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL 
key             : NULL 
key_len         : NULL
Extra           : Using where -- 조건이 테이블 스캔 단계에서 평가됨
```
```sql
| -> Filter: (t1.c1 = 10)  (cost=0.35 rows=1) (actual time=0.0189..0.0189 rows=0 loops=1)
    -> Table scan on t1  (cost=0.35 rows=1) (actual time=0.0181..0.0181 rows=0 loops=1)
 |
```
### index lookup
```sql
create table t2(c1 int, c2 int, index idx(c1));
explain select * from t2 where c1 = 10;
explain analyze select * from t2 where c1 = 10;
```
```sql
type            : ref -- index lookup (Non Unique Index + `=` 조건)
possible_keys   : idx
key             : idx -- 인덱스 사용
key_len         : 5
Extra           : NULL -- covering index 아님
```
```sql
| -> Index lookup on t2 using idx (c1=10)  (cost=0.35 rows=1) (actual time=0.037..0.037 rows=0 loops=1)
 |
```
### 실행 시간 비교
* Full Table Scan : `0.0189..0.0189`
* Index Lookup : `0.037..0.037`
### 결론
데이터가 없거나 적은 경우, full table scan이 index lookup보다 더 빠른 결과를 확인할 수 있다. 테이블 작고 row가 거의 없는 경우에는 테이블 전체를 순차적으로 읽는 sequential I/O 비용이 극히 작은 반면, index lookup은 row수가 적더라도 추가 접근 단계에서 약간의 랜덤 I/O가 발생하기 때문이다.

따라서, 데이터가 없거나 작은 경우 full table scan이 오히려 더 효율적일 수 있으며 정확한 성능 비교를 위해 충분한 데이터를 통한 추가적인 실험이 필요하다.
## 2. non covering index + 충분한 데이터가 존재하는 경우
### full table scan
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int, c2 int);
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10 from seq;
explain analyze select * from t1 where c1 = 10;
```
```sql
| -> Filter: (t1.c1 = 10)  (cost=50000 rows=50000) (actual time=0.0594..198 rows=1 loops=1)
    -> Table scan on t1  (cost=50000 rows=500000) (actual time=0.0502..181 rows=500000 loops=1)
 |
```
### index lookup
```sql
set session cte_max_recursion_depth = 500000;

create table t2 (c1 int, c2 int, index idx(c1));
insert into t2 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10 from seq;
explain analyze select * from t2 where c1 = 10;
```
```sql
| -> Index lookup on t2 using idx (c1=10)  (cost=0.35 rows=1) (actual time=0.0238..0.0264 rows=1 loops=1)
 |
```
### 실행 시간 비교
* Full Table Scan : `0.0594..198`
* Index Lookup : `0.0238..0.0264`
### 결론
충분한 데이터가 삽입된 환경에서 index lookup의 실행시간이 full table scan보다 훨씬 빠르다.

index lookup은 조건에 맞는 row만 찾아서 clustered index를 통해 필요한 데이터에 접근하므로, 전체 테이블을 순차적으로 읽는 full table scan에 비해 I/O와 처리 비용이 크게 줄어든다. 따라서 대량 데이터 조회 환경에서는 적절한 인덱스를 활용한 index lookup이 성능 향상에 매우 효과적이다.


