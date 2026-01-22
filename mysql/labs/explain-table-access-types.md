# 목차
* 테이블 접근 방식 비교
    * Full Table Scan
    * Index Lookup
    * Index Full Scan
    * Index Range Scan
* 인덱스 접근 방식 비교
    * Covering index
    * Back Lookup
    * 정말로 covering index가 더 빠를까?
* 정렬 방식 비교 (order by)
    * Index-Driven Sort
    * Using filesort
<br>

# 테이블 접근 방식 비교
* 참고) [→ explain에 대한 자세한 내용 보기](../explain.md)
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
* 참고) [→ explain에 대한 자세한 내용 보기](../explain.md)
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
## 정말로 covering index가 더 빠를까?
* 실제 많은 데이터를 삽입한 후 조회 쿼리에서 성능 차이가 나는지 확인해보자
* 참고) [→ explain analyze 에 대한 자세한 내용 보기](../explain.md)
### 데이터 삽입
```sql
set session cte_max_recursion_depth = 500000; -- 재귀의 최대 반복 횟수 설정

insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n*10, n*100 from seq;
```
### 실행 결과 확인
#### Back Lookup
```sql
explain analyze select * from t1 where c2 between 100000 and 200000;
```
```sql
| -> Index range scan on t1 using idx1 over (100000 <= c2 <= 200000), with index condition: (t1.c2 between 100000 and 200000)  (cost=4501 rows=10001) (actual time=1.33..10.5 rows=10001 loops=1)
 |
```
* `Index range scan on t1 using idx1`
    * idx1 인덱스를 사용하여 index range scan 수행
* `over (100000 <= c2 <= 200000)`
    * 인덱스 탐색 시작 지점 : 100000
    * 인덱스 탐색 종료 지점 : 200000
* `with index condition: (t1.c2 between 100000 and 200000)`
    * ICP 적용
    * where 조건이 인덱스 스캔 단계에서 처리됨
#### Covering Index
```sql
explain analyze select c1, c2 from t1 where c2 between 100000 and 200000;
```
```sql
| -> Filter: (t1.c2 between 100000 and 200000)  (cost=2003 rows=10001) (actual time=0.159..3.19 rows=10001 loops=1)
    -> Covering index range scan on t1 using idx1 over (100000 <= c2 <= 200000)  (cost=2003 rows=10001) (actual time=0.157..2.61 rows=10001 loops=1)
 |
```
* `Covering index range scan on t1 using idx1`
    * idx1을 covering index로 사용하여 index range scan 수행
* 왜 ICP가 아닌 `Filter`로 나왔을까?
    * `c2 between ...` 조건은 이미 index range scan의 범위 조건으로 처리됨
    * Covering index이므로 추가적인 row 조건 평가나 테이블 접근이 필요 없음
    * 이 경우, MySQL은 조건을 `with index condition`으로 표현하지 않고, 상위에 `Filter` 노드로 표현
        * `Filter`는 형식 상 존재할 뿐, 비용 증가 없음
        * 단, 결과 반환 오버헤드 존재
* `range scan` 노드와 `Filter` 노드의 cost가 동일
* actual time이 `0.157..2.61` 에서 `0.159..3.19`로 살짝 증가
    * 결과 반환 오버헤드로 인한 시간 증가
### 결과 비교
* Back Lookup actual time : `1.33..10.5`
* Covering Index actual time : `0.159..3.19`

→ `actual time`을 비교해보면 전체 처리 완료 시간이 Covering Index가 약 3배 더 빠르게 측정되었다.
### 결론
`Back Lookup`의 경우, 인덱스 스캔 이후 조건에 만족하는 모든 row에 대해 pk를 사용하여 실제 데이터를 재탐색하는 과정이 추가된다. 이로 인해 대상 row 수가 많아질수록 테이블 랜덤 접근 비용이 누적되어 실행 시간이 증가한다.

`Covering Index`의 경우, 조회에 필요한 모든 컬럼을 인덱스만으로 처리하므로 테이블 접근이 발생하지 않으며, 동일한 scan 조건에서도 훨씬 낮은 실행 시간으로 결과를 반환할 수 있다.
<br>

# 정렬 방식 비교 (ORDER BY)
* Index-Driven Sort
    * `인덱스`를 사용한 정렬
    * 인덱스는 이미 정렬된 상태이므로 추가적인 작업이 필요하지 않음
    * 인덱스 순서대로 읽으면 자동으로 정렬된 상태
    * 발생 조건
        * order by 컬럼들이 인덱스에 포함되어 있어야 함
        * order by 순서와 인덱스 컬럼 순서가 일치해야 함
        * 복합 인덱스에서 선행 컬럼에 범위 조건이 사용될 경우, 후행 컬럼들의 정렬이 깨짐
            * 범위 조건 컬럼을 기준으로 여러 그룹으로 나뉨
            * 후속 컬럼들은 각 그룹 내에서만 정렬되어 있음
            * 전체 결과에 대해 정렬된 상태가 아니라서 추가 정렬 작업(`Using filesort`) 발생
    * 성능 상 유리한 경우
        * covering index인 경우
        * LIMIT이 있는 경우
* Using filesort
    * 인덱스를 활용하지 못해서 MySQL이 `별도의 정렬 작업` 수행
        * 메모리 또는 디스크 기반 정렬
        * 필요 시, 임시 테이블 사용
    * 대량 데이터일수록 비용이 많이 들고 느림
    * 발생 조건
        * order by 컬럼이 인덱스에 없는 경우
        * order by와 where 조건이 인덱스 순서를 깨뜨릴 경우
        * 복잡한 조합이나 함수가 포함된 order by
## 기본 ORDER BY (`ASC`)
* 인덱스를 전체 스캔(`index full scan`)하면서 이미 정렬된 인덱스 순서를 사용해 정렬 처리
```sql
create table t1 (c1 int, index idx1(c1));
insert into t1 values (1), (2), (1), (3), (1);

explain select * from t1 order by c1;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : index -- index full scan
possible_keys   : NULL
key             : idx1 -- 인덱스 사용
key_len         : 5
ref             : NULL
rows            : 5
filtered        : 100.00 
Extra           : Using index
```
```sql
+------+
| c1   |
+------+
|    1 |
|    1 |
|    1 |
|    2 |
|    3 |
+------+
5 rows in set (0.00 sec)
```
### +) 인덱스가 걸려있지 않은 컬럼이 추가되면 어떻게 될까?
* 조회 컬럼이 `(c1, c2)`이므로 인덱스(`idx1`)만으로는 조회 불가
* 인덱스를 사용하면 back lookup 발생
* 옵티마이저는 인덱스보다 테이블을 전체 scan하는 것이 더 낫다고 판단
```sql
create table t2 (c1 int, c2 int, index idx1(c1));
insert into t2 values (1, 100), (2, 200), (1, 150), (3, 300), (1, 400);

explain select * from t2 order by c1;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t2
partitions      : NULL
type            : ALL -- full table scan
possible_keys   : NULL
key             : NULL -- 실제로 인덱스를 사용하지 않음
key_len         : NULL
ref             : NULL
rows            : 5
filtered        : 100.00 
Extra           : Using filesort -- MySQL이 별도의 정렬 작업 수행
```
* 참고) `possible_keys=NULL` 인 이유?
    * 인덱스가 있는데 단지 사용하지 않은 거라면 possible_keys에 목록은 출력되어야 하지 않나?
        * possible_keys는 where/join 조건을 기준으로 사용할 수 있는 인덱스 후보 목록을 출력
        * order by 전용 인덱스는 표시되지 않을 수 있음
## 역방향 ORDER BY (`DESC`)
* InnoDB는 B+Tree를 역순으로 읽을 수 있음
    * B+Tree의 leaf노드는 양방향 linked list로 연결되어 있음
```sql
explain select * from t1 order by c1 desc;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : index -- index full scan
possible_keys   : NULL 
key             : idx1 -- 인덱스 사용
key_len         : 5
ref             : NULL
rows            : 5
filtered        : 100.00 
Extra           : Backward index scan; Using index -- filesort 안 생김
```
## WHERE + ORDER BY
### 인덱스 순서를 지키는 경우 (정렬 유지)
* `c1=1` 조건에서 인덱스 첫 컬럼을 `c1`으로 고정
* 그 상태에서 `c2`는 이미 정렬되어 있음
* 추가 정렬 필요 없음
```sql
create table t1 (c1 int, c2 int, index idx1(c1, c2));
insert into t1 values (1, 100), (2, 200), (1, 150), (3, 300), (1, 400);

explain select * from t1 where c1 = 1 order by c2;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : ref -- index lookup (Non Unique Index + `=` 조건)
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5
ref             : const
rows            : 3
filtered        : 100.00 
Extra           : Using index -- filesort 안 생김
```
* 참고) `key_len=5` 인 이유?
    * 복합 인덱스를 구성하는 컬럼 c1, c2가 모두 사용되었는데 왜 5인가?
        * 인덱스 전체 크기 = c1(4) + c2(4) + null flag(1) = 9 byte
        * `key_len`는 실제 옵티마이저가 사용한 인덱스 컬럼까지의 길이를 출력
        * 즉, c1 컬럼까지만 사용되고 c2 컬럼은 사용되지 않음
            * c1 컬럼까지만 탐색해도 조건 처리 및 정렬이 가능함
### where 조건이 정렬을 깨는 경우
* `c1>1` 범위 조건 때문에 인덱스는 `c1`까지 사용
* 조건 결과 집합은 `c2` 기준으로 정렬되어 있지 않음
    * 추가 정렬 작업은 `filesort` 사용
```sql
explain select * from t1 where c1 > 1 order by c2;
```
```sql
id              : 1
select_type     : SIMPLE
table           : t1
partitions      : NULL
type            : range -- index range scan
possible_keys   : idx1 -- where 조건을 처리할 수 있는 인덱스 후보
key             : idx1 -- 인덱스 사용
key_len         : 5 -- c1까지만 인덱스 사용, c2는 인덱스 탐색에 사용되지 않음
ref             : NULL
rows            : 2
filtered        : 100.00 
Extra           : Using where; Using index; Using filesort -- 인덱스 일부 사용, 정렬은 filesort 사용, back lookup 없음
```
## ORDER BY + LIMIT
## Using temporary + using filesort