# 목차
* EXPLAIN 결과 해석
* 테이블 접근 방식 비교 (ALL / ref / index)
    * Full Table Scan
    * Index Lookup
    * Index Full Scan
* 요약
<br>

# EXPLAIN 결과 해석
* id
    * 쿼리 내 `select` 단위의 실행 순서
    * 값이 클수록 먼저 실행
    * 단일 쿼리는 보통 `1`
* select_type
    * select의 종류
        * `SIMPLE` : 서브쿼리 없는 단순 select
        * `PRIMARY` : 최상위 select
        * `SUBQUERY` : where 절 안의 서브쿼리
        * `DERIVED` : from 절의 서브쿼리 (임시 테이블)
        * `UNION` : union의 두 번째 이후 select
* table
    * 접근 대상 테이블 명
    * 서브쿼리나 derived table이면 별칭으로 표시
* partitions
    * 접근하는 파티션 정보
    * 파티션 테이블이 아니면 `NULL`
    * 파티션 pruning 여부 확인용
* type
    * 테이블 접근 방식
        * `ALL` : Full Table Scan
        * `index` : Index Full Scan
        * `range` : Index Range Scan
        * `ref` : Non Unique Index + `=` 조건 (여러 row 가능)
        * `eq_ref` : PK/Unique Index + `=` 조건 (조인에서만 등장, row 1개 보장)
        * `const` : PK/Unique Index + 상수 비교
        * `system` : 테이블에 row가 딱 1개만 존재
* possible_keys
    * 사용 가능하다고 판단한 인덱스 목록
    * 실제로 사용하지 않아도 표시될 수 있음
* key
    * 실제로 사용한 인덱스
    * `NULL`이면 인덱스 미사용
* key_len
    * 사용한 인덱스의 byte 길이
    * 복합 인덱스에서 어떤 컬럼까지 사용했는지 추적 가능
    * 컬럼이 `null`을 허용하는 경우, null 플래그(1 byte) 추가
        * `not null`인 경우 추가되지 않음
* ref
    * 인덱스 비교 대상 값
    * 주로 조인 조건에서 의미 있음
    * `const` 일 경우, 상수와 비교
* rows
    * 옵티마이저가 읽을 것으로 예상한 row 수
    * 실제 결과 row수 아님
    * 성능 튜닝 시, 인덱스 전/후 `rows` 감소가 핵심 지표가 됨
* filtered
    * where 조건을 통과할 것으로 옵티마이저가 예상한 row 비율 (%)
    * 통계 기반의 추정값이므로 절대값 자체는 신뢰하기 어려움
    * **값의 크기보다 변화 추세를 해석하는 것이 중요**
    * `rows * filtered / 100 = 결과 row 추정`
* Extra
    * 부가적인 실행 정보
        * `Using where` : 스토리지 엔진 이후 조건 필터
        * `Using index` : covering index
        * `Using filesort` : 정렬을 인덱스로 못함
        * `Using temporary` : 임시 테이블 사용
        * `Using index condition` : ICP 적용
        * `Backward index scan` : 인덱스 역순 스캔
<br>

# 테이블 접근 방식 비교 (ALL / ref / index)
## Full Table Scan
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
Extra           : Using index -- covering index 사용
```
* `filtered`의 값이 10.00 → 100.00 으로 변함
    * 인덱스에서 where 조건을 만족하는 row를 걸렀음을 의미
## Index Full Scan
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
<br>

# 요약
* `EXPLAIN`은 MySQL 옵티마이저가 테이블에 어떻게 접근할지 보여주는 실행 계획 도구이다.
* `type=ALL`은 **Full Table Scan**으로, 테이블 전체 row를 읽는 가장 비효율적인 접근 방식이다.
* `type=ref`는 **Index Lookup**으로, 인덱스를 이용해 필요한 row만 탐색하는 가장 효율적인 방식이다.
* `type=index`는 **Full Index Scan**으로, 인덱스를 사용하지만 전체를 순차 스캔하므로 필터링 효과가 없다.
* `rows`는 실제 결과 row 수가 아니라, 옵티마이저가 읽을 것으로 예상한 추정값이다.
* `filtered`는 읽은 row 중 where 조건을 통과할 것으로 예상되는 비율로, 절대값보다 변화 추세를 보는 것이 중요하다.
* 일반적으로 `type`이 `ALL → index → ref` 순으로 성능이 개선되며, 튜닝의 1차 목표는 `ALL`을 `ref` 이상으로 낮추는 것이다.
