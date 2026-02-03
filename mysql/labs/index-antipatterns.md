# 목차
* INDEX를 타지 않는 쿼리 패턴
* INDEX 효율이 떨어지는 쿼리 패턴
<br>


# INDEX를 타지 않는 쿼리 패턴
* 인덱스가 존재하더라도 MySQL 옵티마이저가 인덱스를 사용할 수 없거나, 아예 후보에서 제외하는 쿼리 패턴
* 단순히 인덱스를 사용하여 쿼리를 수행한 것이 아닌, **조건 기반 인덱스 탐색이 이루어 진 경우**를 말함
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
### 1. 컬럼에 함수를 사용한 경우
* 컬럼에 함수가 적용되면 인덱스 정렬 기준과 비교 기준이 달라짐
* B-Tree 탐색 시작 지점을 계산할 수 없음
* `table/index full scan`이 선택됨
* **기본적으로 저장할 때 비교 상황을 고려하여 적절한 형태로 저장해서 해결 할 수 있음**
#### 1) 문자열 함수 - `lower` / `upper`
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
#### 2) 문자열 함수 - `trim` / `ltrim` / `rtrim`
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
#### 3) 문자열 함수 - `substring` / `substr` / `left` / `right`
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

### 2. 타입 불일치 (암시적 형 변환)
[→ 암시적 형 변환에 대한 내용 자세히 보기](../implicit-type-conversion.md)
### 3. 산술 연산
## 인덱스 탐색 시작 지점을 특정할 수 없는 경우
### 잘못된 LIKE 패턴
### 복합 인덱스 선두 컬럼 미사용
### OR 조건 남용
## 인덱스 사용 효용이 거의 없는 경우
### 부정 조건 사용
<br>

# INDEX 효율이 떨어지는 쿼리 패턴
## 범위 조건 이후 컬럼 사용
## SELECT * 사용 (Covering 불가)
## 낮은 Selectivity 컬럼
## IS NULL / IS NOT NULL
## IN 절에 값이 너무 많을 경우
## ORDER BY / GROUP BY 컬럼 불일치
## LIMIT + OFFSET 페이징