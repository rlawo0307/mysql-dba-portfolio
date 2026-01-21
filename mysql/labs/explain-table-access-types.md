# 목차
* 테이블 접근 방식 비교
    * Full Table Scan
    * Index Lookup
    * Index Full Scan
    * Index Range Scan
* 인덱스 접근 방식 비교
    * Covering index
    * Back Lookup
* 요약
<br>

# 테이블 접근 방식 비교
* 참고) [→ explain에 대한 자세한 내용 보기](../explain.md)
## Full Table Scan
* 테이블의 모든 행을 처음부터 끝까지 읽는 방식
* 인덱스를 사용하지 않고 테이블을 통째로 탐색
```sql
create table t1(c1 int);

insert into t1(c1) with recursive seq as (select 1 as n union all select n+1 from seq where n < 100) select n from seq;

explain select * from t1 where c1 = 10;
```
```sql
id              : 1 
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : ALL -- full table scan
possible_keys   : NULL -- 사용 가능한 인덱스 없음
key             : NULL -- 인덱스 미사용
key_len         : NULL
ref             : NULL
rows            : 100 -- 전체 row를 읽음 (추정)
filtered        : 10.00
Extra           : Using where
```
## Index Lookup
* 인덱스를 이용해 필요한 row 위치를 찾는 접근 방식
* 특정 값으로 인덱스를 타고 바로 점프
* `type = range / ref / eq_ref / const` 등
* **Index Lookup ≠ Back Lookup** 
```sql
create index idx1 on t1(c1);
explain select * from t1 where c1 = 10;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : ref --  Non Unique Index + `=` 조건
possible_keys   : idx1 -- 사용 가능한 인덱스 목록 : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5 -- int(4 bytes) + NULL 플래그(1 byte)
ref             : const -- 인덱스와 상수를 비교
rows            : 1 -- 1개의 row를 읽음 (추정)
filtered        : 100.00
Extra           : Using index
```
* `filtered`의 값이 10.00 → 100.00 으로 변함
    * 인덱스에서 where 조건을 만족하는 row를 걸렀음을 의미
## Index Full Scan
* 인덱스의 모든 엔트리를 처음부터 끝까지 순차적으로 읽는 방식
* 조건이 없거나 인덱스를 통한 전체 정렬이 필요한 경우 발생
* 커버링 인덱스일 경우, 테이블 접근 없이 인덱스만으로 처리할 때 발생
```sql
explain select c1 from t1;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : index -- index full scan
possible_keys   : NULL -- where 조건이 없어서 후보 인덱스를 고려할 필요 없음
key             : idx1 -- 인덱스 사용
key_len         : 5
ref             : NULL
rows            : 100 -- 전체 row를 읽음 (추정)
filtered        : 100.00 -- 필터링 조건이 없으므로 모든 row가 조건을 통과
Extra           : Using index -- covering index 사용
```
* 인덱스를 사용했지만 조건 없이 인덱스 전체를 스캔
* 테이블 접근 없이 `covering index`로 결과 처리
    * 테이블보다 인덱스 페이지가 작기 때문에 디스크 I/O 관점에서 `ALL`보단 나음
    * 필터링 효과가 없기 때문에 `ref`보다는 훨씬 나쁨
## Index Range Scan
* 인덱스의 일부 범위만 탐색하는 방식
```sql
explain select * from t1 where c1 between 10 and 20;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : range -- index range scan
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5
ref             : NULL
rows            : 11
filtered        : 100.00 -- 불필요한 row 재검사 없을 것이라 추정
Extra           : Using where; Using index
```
<br>

# 인덱스 접근 방식 비교
## Covering Index
## Back Lookup 
<br>

# 요약
* `rows`는 실제 결과 row 수가 아니라, 옵티마이저가 읽을 것으로 예상한 추정값이다.
* `filtered`는 읽은 row 중 where 조건을 통과할 것으로 예상되는 비율로, 절대값보다 변화 추세를 보는 것이 중요하다.
* 일반적으로 `type`이 `ALL → index → ref` 순으로 성능이 개선되며, 튜닝의 1차 목표는 `ALL`을 `ref` 또는 `range` 이상으로 낮추는 것이다.
