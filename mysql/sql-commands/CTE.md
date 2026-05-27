# CTE(Common Table Expression)
* 하나의 SQL 문 안에서 이름을 붙여 참조하는 논리적 결과 집합
* `with` 절로 정의
* 뒤에 오는 select / insert / update / delete 문이 참조할 수 있음
* 해당 SQL 문에서만 유효하며 SQL 실행이 끝나면 사라짐
* 실제 table / view / permanent object가 아님
* MySQL 8.0 이상에서 지원
* 장점
    * 복잡한 subquery를 분리하여 가독성을 향상
    * 단계별 query 분리 가능
    * 같은 SQL 문 안에서 query 재사용 가능
    * recursive query 가능
```sql
with cte_name as (select ... ) select ...;
with cte_name1 as (...), cte_name2 as (...) select ...;
```
## 처리 방법
* CTE는 항상 temp table을 만드는 것은 아님
* optimizer가 결정한 처리 방식에 따라 temp table 생성 여부가 달라짐
* explain 실행 계획을 통해 optimizer가 CTE를 어떻게 처리했는지 간접적으로 추적 가능
### Merge(inline) 방식
* CTE를 실제로 만들지 않고 query 내부로 펼치는 방식
* temp table 없음
```sql
-- 원본 query
with cte as (select * from t1 where c1 = 10) select * from cte where c2 = 20;

-- optimizer가 내부적으로 이런 형태로 처리 가능
select * from t1 where c1 = 10 and c2 = 20;
```
### Materialization 방식
* CTE 결과를 임시 저장하여 main query에서 사용하는 방식
* memory 또는 disk 사용(I/O 발생)
* CTE 결과가 클수록 속도가 저하될 수 있음
## recursive CTE
* CTE가 이전 iteration에서 생성된 결과를 참조하여 반복적으로 결과를 확장하는 query
* 무한 루프에 빠질 수 있으므로 종료 조건에 항상 신경써야 함
* `cte_max_recursion_depth`
    * MySQL에서 recursive CTE가 최대 몇 번까지 재귀 반복될 수 있는지 제한하는 시스템 변수
    * recursive CTE가 무한 루프 도는 것을 방지하기 위한 안전장치
    * anchor query는 포함되지 않음
    * recursive query 반복 횟수만 카운트
    * 설정 값을 초과하면 에러 출력(`ERROR 3636`)
### 문법
```sql
with recursive cte as
(
    anchor query -- 초기 시작 값
    union all
    recursive query -- CTE를 참조하여 다음 결과 생성
)
select * from cte;
```
### 예시
```sql
set session cte_max_recursion_depth = 5000;

with recursive seq as
(
    select 1 as n -- n에 초기 값 1을 설정
    union all
    select n+1 from seq where n < 5000 -- 현재 결과 중 n < 5000인 row를 기준으로 다음 값을 생성
)
select * from seq;
```
## CTE vs Subquery
### CTE
```sql
with cte as (select * from t1) select * from cte;
```
### Subquery
```sql
select * from (select * from t1) x;
```
<center>

|              |가독성         |재사용        |재귀|유지보수|
|--------------|:------------:|:-----------:|:-:|:-----:|
|**CTE**       |좋음           |같은 SQL문 내에서 가능         |가능|좋음    |
|**Subquery**  |복잡해질 수 있음|중복 작성 필요|불가|불편    |
</center>

#### ※ 단, 성능은 무조건 CTE가 빠른것은 아님 (optimizer 처리에 따라 다름)
