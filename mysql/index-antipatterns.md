# INDEX를 타지 않는 쿼리 패턴
* 인덱스가 존재하더라도 MySQL 옵티마이저가 인덱스를 사용할 수 없거나, 아예 후보에서 제외하는 쿼리 패턴
* "index를 탄다"
    * 단순히 인덱스를 사용하여 쿼리를 수행한 것이 아님
    * **조건 기반 인덱스 탐색이 이루어 진 경우**를 말함
* 인덱스를 타는 경우
    * Index Lookup (`ref`, `eq_ref`, `const`)
    * Index Range Scan (`range`)
* 인덱스를 타지 않는 경우
    * Table Full Scan (`ALL`)
    * Index Full Scan (`index`)
        * 인덱스를 사용한 full scan일뿐, 조건 기반 인덱스 탐색을 실패한 결과임
## 컬럼 값이 변형된 경우
* 인덱스에는 저장된 값 그대로 정렬되어 있음
* 컬럼 값이 변형되는 순간 B-Tree 탐색 불가
* `table/index full scan`이 선택됨
* **기본적으로 저장할 때 비교 상황을 고려하여 적절한 형태로 저장해서 해결 할 수 있음**
### 1. 컬럼에 함수를 사용한 경우
#### ex) 문자열 대소문자 구분 - `lower` / `upper`
* 해결 방법) 대소문자 구분 없는 `collation` 사용
    * 컬럼 정의에 설정된 collation과 완전히 동일해야 함
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 varchar(100), index idx1(c1));
insert into t1 with recursive seq as
(
    select 1 as n union all select n+1 from seq where n < 500000
) select concat('val_', n) from seq;
insert into t1 values ('aaa');

explain select * from t1 where lower(c1) = 'aaa'; -- type=index, key=idx1
```
```sql
show full columns from t1; -- Collation : utf8mb4_0900_ai_ci
explain select * from t1 where c1 collate utf8mb4_0900_ai_ci = 'aaa'; -- type=ref, key=idx1
```
#### ex) 문자열 공백 제거 및 변형 - `trim` / `ltrim` / `rtrim`
* 해결 방법) `generated column`을 생성하여 해당 컬럼에 인덱스 생성
    * 테이블에 실제로 저장되거나 계산 시점에 자동으로 만들어지는 가상의 컬럼
    * 기본적으로 다른 컬럼의 값을 기반으로 계산된 결과를 저장하는 컬럼
    * 직접 값을 넣지 않아도 MySQL이 자동으로 값을 생성해줌
    * 종류
        * `Stored Generated Column`
            * 계산 결과를 테이블에 실제 저장
            * 인덱스 생성 가능
        * `Virtual Generated Column`
            * 계산 결과를 저장하지 않고 조회 시점에 계산
            * 인덱스 생성 불가
```sql
create table t1(c1 varchar(100), index idx1(c1));
explain select * from t1 where trim(c1) = 'aaa'; -- type=index, key=idx1
```
```sql
alter table t1 add gc1 varchar(100) generated always as (trim(c1)) stored;
alter table t1 add index idx2(gc1);
explain select * from t1 where gc1 = 'aaa'; -- type=ref, key=idx2
```
#### ex) 부분 문자열 추출 - `substring` / `substr` / `left` / `right`
* 해결 방법 1) 범위 조건 사용
* 해결 방법 2) `like` 사용
* 해결 방법 3) `generated column`을 생성하여 해당 컬럼에 인덱스 생성
    * 원본 컬럼의 prefix를 generated column으로 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 varchar(100), index idx1(c1));
insert into t1 with recursive seq as
(
    select 1 as n union all select n+1 from seq where n < 500000
) select concat('val_', n) from seq;
insert into t1 values ('aaa');

explain select * from t1 where substr(c1, 1, 3) = 'aaa'; -- type=index, key=idx1
```
```sql
explain select * from t1 where c1 >= 'aaa' and c1 < 'aaab'; -- type=range, key=idx1
explain select * from t1 where c1 like 'aaa%'; -- type=range, key=idx1

alter table t1 add gc1 varchar(100) generated always as (substr(c1, 1, 3)) stored;
alter table t1 add index idx2(gc1);
explain select * from t1 where gc1 = 'aaa'; -- type=ref, key=idx2
```
#### ex) 숫자, 날짜/시간 함수 - `round` / `floor` / `ceiling` / `date` / `year` / `month` / `day` 등
* 해결 방법) 범위 조건 사용
```sql
create table t1 (c1 int, c2 date, index idx1(c1), index idx2(c2));
explain select * from t1 where round(c1) = 100; -- type=ALL
explain select * from t1 where year(c1) = 2024; -- type=ALL
```
```sql
explain select * from t1 where c1 >= 99.5 and c1 < 100; -- type=range, key=idx1
explain select * from t1 where c2 >= '2024-01-01' and c2 < '2025-01-01'; -- type=range, key=idx2
```
### 2. 타입 불일치 (암시적 형 변환)
* [→ 암시적 형 변환에 대한 내용 자세히 보기](../implicit-type-conversion.md)
* 해결 방법 1) 타입 일치시키기
* 해결 방법 2) 명시적 형 변환 사용
* 해결 방법 3) 컬럼 타입 변경을 고려해보기
```sql
create table t1 (c1 int, c2 varchar(6), index idx1(c1), index idx2(c2), index idx3(c1, c2));

explain select c1 from t1 where c1 = '123'; -- type=ref, key=idx1
explain select c2 from t1 where c2 = 123; -- type=index, key=idx2
explain select * from t1 where c1 = c2; -- type=index, key=idx3
```
* `where c1 = '123'`
    * 일반적으로는 값을 컬럼 타입으로 변환하는 것이 원칙
    * `'123'`이 int 타입으로 변환된 후 비교
    * `idx1` 사용 가능
* `where c2 = 123`
    * 실제 동작에서는 옵티마이저가 값이 아닌 컬럼의 타입을 변환하는 것이 좋다고 판단할 수 있음
    * 값(`123`)이 아닌 컬럼(`c2`)에 암시적 형 변환 수행
    * 인덱스 무력화
* `where c1 = c2`
    * 타입 우선순위에 의해 `c2`가 int 타입으로 변환
    * 복합 인덱스에서는 키 중 하나라도 동적 계산이 되면 탐색의 시작 지점을 알 수 없음
    * 조건 기반 인덱스 탐색 불가능   
### 3. 산술 연산
* 해결 방법) 연산을 값 쪽으로 이동
```sql 
create table t1 (c1 int, index idx1(c1));
explain select * from t1 where c1 + 1 = 100; -- type=index, key=idx1
```
```sql
explain select * from t1 where c1 = 100-1; -- type=ref, key=idx1 
```
## 인덱스 탐색 시작 지점을 특정할 수 없는 경우
* 인덱스 탐색 시작 지점을 알 수 없다면 조건 기반 인덱스 탐색 불가
* `table/index full scan`이 선택됨
### 와일드카드로 시작하는 LIKE 패턴
* 문자열의 앞부분이 불확실함
* 조건 기반 인덱스 탐색 불가능
* 와일드카드 `%`, `_` 모두 동일하게 적용
```sql
create table t1(c1 varchar(100), index idx1(c1));
explain select * from t1 where c1 like '%111'; -- type=index, key=idx1
explain select * from t1 where c1 like '1%11'; -- type=range, key=idx1
```
### 서로 다른 컬럼에 대한 OR 조건
* or 로 연결된 조건들은 각각 독립적인 탐색 조건으로 평가됨
* 서로 다른 컬럼에 대한 조건은 단일 B-Tree 탐색 경로로 합칠 수 없음
* 하나의 인덱스로 n 개의 조건을 동시에 만족시키는 탐색 불가능
* 옵티마이저는 `full scan` 또는 `index merge`를 선택할 수 있음
```sql
create table t1 (c1 int, c2 int, index idx1(c1, c2));
explain select * from t1 where c1 = 10 or c2 = 20; -- type=index, key=idx1
```
<br>

# INDEX 효율이 떨어지는 쿼리 패턴
## 부정 조건 사용
* `!=`, `<>`, `NOT IN`, `NOT LIKE`, `IS NOT NULL` 등
* 부정 조건은 제외할 값만 알려줌
* 탐색해야 할 범위를 특정할 수 없음
* 포함해야 할 값을 기준으로 탐색 범위를 특정할 수 없음
* `null`이 포함될 경우
    * 내부적으로 추가적인 평가 로직이 필요
    * 실행 계획이 더 복잡해 질 수 있음
* 인덱스를 써도 걸러낼 것이 거의 없음
* 즉, 인덱스를 사용할 수 없는 것이 아니라 비용 대비 효과가 없어서 옵티마이저가 사용하지 않는 것
## 범위 조건 이후 컬럼 사용
* 복합 인덱스에서 범위 조건이 등장하는 순간, 그 뒤 컬럼은 탐색 키로 사용할 수 없음
    * 범위 조건은 탐색 시작 지점을 하나의 값으로 고정할 수 없음
    * 이후 컬럼의 정렬 위치를 예측할 수 없음
* 범위 조건 이후의 조건은 인덱스 탐색 이후 Filter 단계에서 처리
```sql
create table t1 (c1 int, c2 int, c3 int, index idx1(c1, c2, c3));
insert into t1 (c1, c2, c3) with recursive seq as
(
    select 1 as n union all select n + 1 from seq where n < 100
) select s1.n as c1, s2.n as c2, s3.n as c3 from seq s1 cross join seq s2 cross join seq s3;

select * from t1 where c1 = 10 and c2 > 20 and c3 = 30;
```
```sql
type            : range
key             : idx1
key_len         : 10 -- (c1 + null flag) + (c2 + null flag) = 5 + 5 = 10
Extra           : Using where; Using index -- 인덱스 탐색 & 필터링 수행
```
```sql
| -> Filter: ((t1.c3 = 30) and (t1.c1 = 10) and (t1.c2 > 20))  (cost=3133 rows=1562) (actual time=0.0557..6.87 rows=80 loops=1)
    -> Covering index range scan on t1 using idx1 over (c1 = 10 AND 20 < c2)  (cost=3133 rows=15616) (actual time=0.0354..6.41 rows=8000 loops=1)
 |
```
## SELECT * 사용 (Covering 불가)
* covering index가 아닌 secondary index의 경우 pk가 아닌 나머지 컬럼을 읽기 위해 back lookup 수행
* 조회되는 row 수가 많을 수록 비용 급증
* 차라리 full scan이 낫다고 생각된다면 인덱스 탐색 포기
## 낮은 Selectivity 컬럼
* 값의 종류가 적을수록 selectivity가 낮음
    * `selectivity = 조건을 만족하는 row 수 / 전체 row 수`
* selectvitiy가 낮을수록
    * 걸러지는 row 수가 적음
    * 인덱스 효용이 떨어짐
    * 인덱스 탐색 + back lookup 비용이 full scan 비용보다 커질 수 있음
    * 비용 대비 효과가 낮다고 판단된다면 인덱스 탐색 포기
## IN 절에 값이 너무 많을 경우
* `in` 절은 내부적으로 여러 개의 동등 조건(=)으로 처리됨
    *  `c1 IN (1,2,3)` ≒ `c1 = 1 OR c1 = 2 OR c1 = 3`
* 각 값에 대해 인덱스 탐색이 반복적으로 수행됨
    * 조건 수만큼 인덱스 탐색 발생
    * covering index가 아니라면 각 탐색마다 back lookup 발생
    * 값이 많아질수록 누적 비용이 커짐
* 비용 대비 효과가 낮다고 판단된다면 인덱스 탐색 포기
## ORDER BY 컬럼 불일치
* 인덱스 순서와 정렬 순서가 완전히 일치할 때만 추가 정렬(filesort) 없이 인덱스로 정렬 처리 가능
* 순서가 같더라도 방향(`asc`, `desc`)이 섞여있다면 인덱스 정렬 사용 불가
* [→ 인덱스를 사용한 정렬에 대한 내용 자세히 보기](explain-table-access-types.md)
## GROUP BY 컬럼 불일치
* 인덱스 순서와 group by 컬럼 구성 및 순서가 일치하는 경우
    * 같은 값들이 인덱스 상에서 연속해서 등장
    * 추가 정렬이나 임시 테이블 없이 그룹화 가능
* 인덱스 순서와 group by 컬럼 구성 및 순서가 일치하지 않는 경우
    * 같은 값이 흩어져서 존재하므로 그룹 경계를 알 수 없음
    * 추가적인 정렬 작업 필요
* group by 컬럼에 함수가 사용된 경우
    * 컬럼 값이 동적 계산되어 인덱스 정렬 기준과 일치하지 않음
    * 인덱스 기반 그룹화 불가
### group by + where 절
* 인덱스 선두 컬럼들이 where 절에서 상수로 고정된 경우, 인덱스와 group by의 컬럼 구성이 다르지만 인덱스 기반 그룹화가 가능
    * where 절 조건에 의해 인덱스 탐색 범위가 고정됨
    * 해당 범위 내에서는 다음 컬럼 값들이 인덱스 상에서 연속해서 등장
```sql
create table t1 (c1 int, c2 int, index idx1(c1, c2));
select c2, count(*) from t1 where c1 = 10 group by c2;
```
```sql
type            : ref -- c1=10 동등조건으로 인덱스 접근
key             : idx1
key_len         : 5 -- c1(4) + null flag(1)
Extra           : Using index
```
```sql
| -> Group aggregate: count(0)  (cost=0.45 rows=1) (actual time=0.0142..0.0142 rows=0 loops=1)
    -> Covering index lookup on t1 using idx1 (c1=10)  (cost=0.35 rows=1) (actual time=0.0129..0.0129 rows=0 loops=1)
 |
```
## LIMIT + OFFSET 페이징
* `offset`은 건너뛰는 것이 아닌 **읽고 버리는 것**
* offset이 커질수록 읽어야 할 row 수가 선형적으로 증가
* 인덱스를 사용하더라도 앞쪽 row를 모두 스캔해야 함
* 뒤 page로 갈수록 비용 대비 효율이 매우 낮음