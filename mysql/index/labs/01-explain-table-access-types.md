# 목차
* 테이블 접근 방식 비교
    * Full Table Scan
    * Index Lookup
    * Index Full Scan
    * Index Range Scan
* 인덱스 접근 방식 비교
    * Covering index
    * Back Lookup
<br>

# 테이블 접근 방식 비교
* [→ explain에 대한 자세한 내용 보기](../optimization/explain.md)
* [→ 각 scan 방식에 대한 개념 자세히 보기](../index/index-basics.md)
## Full Table Scan
* 테이블의 모든 행을 처음부터 끝까지 읽는 방식
* 인덱스를 사용하지 않고 테이블을 통째로 탐색
```sql
create table t1(c1 int, c2 int);

insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 100) select n, n*10 from seq;

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
Extra           : NULL
```
* `filtered`의 값이 10.00 → 100.00 으로 변함
    * 인덱스에서 where 조건을 만족하는 row를 걸렀음을 의미
## Index Full Scan
* 인덱스의 모든 엔트리를 처음부터 끝까지 순차적으로 읽는 방식
* 조건이 없거나, 인덱스를 통한 전체 정렬이 유리하다고 판단될 경우 발생
* 커버링 인덱스일 경우, 테이블 접근 없이 인덱스만으로 처리할 때 발생
```sql
explain select c2 from t1;
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
filtered        : 100.00 -- 인덱스 단계에서 조건이 완전히 처리될 것으로 추정
Extra           : Using index condition -- where 조건이 인덱스 스캔 단계에서 처리됨
```
<br>

# 인덱스 접근 방식 비교
* [→ explain에 대한 자세한 내용 보기](../optimization/explain.md)
## Back Lookup
* secondary index로 row 위치를 찾은 후, pk를 이용해 테이블에서 실제 데이터를 조회하는 방식
* 인덱스에 없는 컬럼을 조회할 때 발생
* `expain` 결과만 보고 back lookup 발생 여부를 확정할 수 없음
* `back lookup`은 `index lookup`의 내부 동작이라 따로 표시되지 않음
* 단, 거의 확실하게 추론 가능
    * back lookup이 발생하지 않는 경우
        * covering index로 조건을 처리한 경우 → `Extra=Using index`
    * back lookup이 발생하는 경우
        * 아래 조건이 동시에 만족한다면 99% 발생했다고 해석
            1. `key != NULL`
            2. `Extra != Using index`
            3. 조회에 필요한 모든 컬럼이 인덱스에 포함되지 않은 경우
```sql
create table t1 (c1 int primary key, c2 int, c3 int, index idx1 (c2));

explain select * from t1 where c2 = 30;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : ref -- Non Unique Index + `=` 조건 (Index Lookup 수행)
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5
ref             : const
rows            : 1
filtered        : 100.00 -- 인덱스 단계에서 조건이 완전히 처리될 것으로 추정
Extra           : NULL -- where 조건은 인덱스 접근 단계에서 완전히 처리됨
```
* **`Extra=NULL` 은 back lookup이 발생했다는 의미가 아님**
    * 표시할 추가 정보가 없다는 의미일 뿐
    * back lookup 여부와 직접적인 관계 없음
* 참고) `Extra=Using where` 역시 back lookup이 발생했다는 의미가 아님
## Covering Index
* 쿼리에 필요한 모든 컬럼을 인덱스만으로 처리하는 접근 방식
* `back lookup`이 발생하지 않음
* InnoDB에서는 secondary index에 포함되는 pk 컬럼도 covering 판단에 포함
    * `secondary index = (key, pk 값)`
```sql
explain select c1, c2 from t1 where c2 = 30;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : ref
possible_keys   : idx1
key             : idx1 
key_len         : 5
ref             : const
rows            : 1
filtered        : 100.00 
Extra           : Using index -- covering index 사용
```
* secondary index인 `idx1`에는 pk 컬럼이 자동으로 포함됨 (InnoDB 기준)
    * `c1`은 pk 컬럼이며, 그 값 자체가 row를 식별하는 pk 값
    * `idx1`은 개념적으로 `(c2, c1)` 구조를 가짐
* 조회에 필요한 모든 컬럼이 인덱스에 존재하므로 테이블 접근 불필요 → `covering index` 사용
<br>