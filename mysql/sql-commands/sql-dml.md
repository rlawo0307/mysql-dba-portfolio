# 데이터 조회하기 (SELECT)
```sql
select 컬럼명1, 컬럼명2, ...
[from 테이블/뷰]
[where 조건]
[group by 컬럼명1, 컬럼명2, ...]
[having 집계조건]
[order by 컬럼명1 [asc|desc], 컬럼명2 [asc|desc], ...]
[limit 개수 offset 시작점];
```
### **쿼리 수행 순서 : `from → where → group by → having → select → order by → limit`**
* select 절
    * 결과로 반환할 컬럼이나 계산된 값을 지정
    * 단순 컬럼 외에 산술 연산, 문자열 함수, 집계 함수 등을 사용할 수 있음
    * 서브 쿼리 사용 가능
        * select 절에 사용한 서브쿼리를 `scalar subquery`라고 함
        * 하나의 값을 반환하는 서브쿼리
        * 주로 특정 컬럼 값이나 계산 결과로 하나의 값이 필요할 때 사용
    * `as 별칭`으로 결과 컬럼의 이름을 지정 가능
    * `distinct` 키워드를 사용하여 중복된 결과 제거 가능
    * asterisk(`*`)를 사용해서 모든 컬럼을 조회할 수 있음
        * 단, 불필요한 컬럼까지 모두 읽어 성능 저하를 유발할 수 있음
* from 절
    * 데이터를 가져올 테이블 또는 뷰 지정
    * 여러 테이블을 조인하여 지정할 수 있음
    * 서브쿼리를 테이블처럼 사용할 수 있음
        * from 절에 사용한 서브쿼리를 `inline view`라고 함
        * 임시 테이블처럼 동작
        * 주로 복잡한 쿼리나 집계, 필터링을 단계별로 처리할 때 사용
        * 복잡한 inline view는 성능에 큰 영향을 줄 수 있으므로 주의
    * from 절 없이 select만 쓸 경우, 상수값을 조회할 수 있지만 테이블 데이터는 조회 불가
* where 절
    * 결과 행을 필터링 (조건 지정)
    * 논리 연산자, 비교 연산자 등 사용 가능
    * **where 절에서는 집계 함수 사용 불가**
        * 집계 결과를 조건으로 걸고 싶다면 `having 절 사용`
    * select 절에서 정의한 별칭(alias)를 where절에서 사용할 수 없음
        * SQL 실행 순서에 의해 where 절이 select 절보다 먼저 수행됨
        * select 절에서 정의한 별칭은 where 절 실행 시점에 존재하지 않음
* group by 절
    * 결과를 지정한 컬럼 기준으로 그룹화
    * 표현식 사용 가능
        * 복잡한 식은 성능 저하를 유발할 수 있음
    * **select 절에 집계 함수가 아닌 컬럼이 있다면 반드시 group by 절에 포함되어 있어야 함**
* having 절
    * 그룹화된 결과에 대한 조건 필터링
        * where 절은 행 단위 필터링
        * having 절은 그룹 단위 필터링
    * where 절과 달리 집계 함수 사용 가능
    * group by 절과 함께 사용해야 의미가 있음
    * having 절에서 집계 함수 없이 컬럼만 쓸 경우, 일반적으로 where 절로 바꾸는 것이 성능에 유리함
* order by 절
    * 결과 정렬
    * 기본 정렬 순서는 오름차순(`asc`)
    * 여러 개의 컬럼을 지정하여 복합 정렬 가능
    * select 절에서 지정한 별칭 사용 가능
    * 정렬 기준으로 집계 함수 지정 불가
        * order by는 이미 집계가 끝난 상태의 결과 컬럼들을 대상으로 정렬 수행
* limit 절
    * 결과 개수 제한 및 offset 지정
    * [→ limit에 대한 내용 자세히 보기](../limit.md)
<br>

# 데이터 삽입 (INSERT)
```sql
insert [options] into 테이블명 [(컬럼명1, 컬럼명2, ...), ...] values (값1, 값2, ...);
```
* 반드시 삽입할 컬럼과 값이 순서대로 일치해야 함
* 컬럼명을 생략하면 테이블 컬럼 순서대로 값을 넣음 (권장 X)
    * 값과 1:1 대응되지 않는 컬럼은 `default` 값 또는 `null` 처리됨
    * `auto_increment` 컬럼일 경우, 자동으로 1씩 증가하는 값이 할당됨
* 삽입하려는 컬럼의 자료형과 값이 일치해야 함
* 제약조건을 반드시 충족해야 함
    * [→ 각 제약조건에에 대한 내용 자세히 보기](../constraint-theory.md)
* options
    * `ignore`
        * 기본적으로 중복 키 충돌 발생 시 오류가 나면서 쿼리가 중단됨
        * ignore 옵션 사용 시, 충돌이 나는 행은 무시하고 나머지 행은 정상 삽입
            * 중복 키 충돌이 아닌 다른 오류들도 무시될 수 있으니 주의 필요
        * 무시된 행에 대한 경고가 남음
            * `show warnings;` 으로 확인 가능
    * `on duplicate key update`
        * 중복 키 충돌 발생 시, 오류를 발생시키지 않고 update 구문 수행
        * 충돌 시 데이터 갱신이 필요한 경우 사용
    * `low_priority`
        * insert 작업의 우선순위를 낮춤
        * 테이블의 다른 읽기 작업이 완료될 때까지 대기
        * 대량 삽입 시 읽기 작업에 미치는 영향을 줄이기 위해 사용
    * `high_priority`
        * insert 작업의 우선순위를 높임
        * 테이블에 쓰기 작업이 머저 수행되도록 강제
```sql
insert into 테이블명 [(컬럼명1, 컬럼명2, ...), ...] select ..  from ..;
```
* 다른 테이블에서 조건에 맞는 데이터를 복사해 올 때 사용
* 삽입 대상 컬럼과 select 절의 컬럼 순서와 개수가 일치해야 함
* 삽입 대상 컬럼과 select 절의 컬럼 데이터 타입이 동일하거나 호환 가능해야 함
* 제약 조건에 유의해야 함
* 대량 데이터 복사나 백업 시 유용 
<br>

# 데이터 수정 (UPDATE)
* 기본적으로 트랜잭션 내에서 동작하므로 롤백 가능
* MySQL에서 기본적으로 `before update`, `after update` 트리거를 지원
```sql
update [options] 테이블명
set 컬럼명1=값1, 컬럼명2=값2, ...
[where 조건]
[order by 컬럼명1 [asc|desc], 컬럼명2 [asc|desc], ...]
[limit 개수];
```
* options : `ignore`, `low_priority`, `high_priority`
* set 절
    * 수정할 컬럼과 새로운 값을 지정
    * 수정 대상 컬럼에 인덱스가 있으면 성능에 영향을 미칠 수 있음
* where 절
    * 수정할 행을 필터링하는 조건 지정
    * 조건을 지정하지 않으면 테이블의 모든 행에 대해 수정되므로 주의 필요
* order by 절
    * 여러 행을 수정할 때, 어떤 순서로 행들을 선택할지 결정
    * limit 절과 함께 사용되며 단독 사용 시 의미 없음
* limit 절
    * 한번에 수정할 행 수를 제한
    * 여러 행이 조건에 맞을 때, 일부만 수정하거나 대량 업데이트 시 영향을 줄이고 싶을 때 사용
    * offset을 지원하지 않음
        * 내부적으로 저장된 특정 위치를 건너뛰고 작업하는 기능 없음
        * where 조건과 order by, limit을 활용해 작업 분할 필요
    * order by 절 없이 사용될 수 있음
<br>

# 데이터 삭제 (DELETE)
* 트랜잭션 내에서 처리되므로 롤백 가능
* `foreign key` 제약조건에 의해 삭제가 제한될 수 있음
* 테이블 전체 데이터 삭제 시, `truncate` 명령어가 더 빠름
    * 단, 롤백 불가
```sql
delete [options] [테이블명]
from 테이블명 [join ..] [join ..] ...
[where 조건]
[order by 컬럼명1 [asc|desc], 컬럼명2 [asc|desc], ...]
[limit 개수];
```
* delete 절
    * options : `ignore`, `low_priority`, `high_priority`
    * 테이블 명
        * 실제 데이터를 삭제할 대상 테이블 명시
        * 테이블명을 명시하지 않으면 from 절에 명시된 첫 번째 테이블에 대해 삭제 수행
            * `delete from t1` → t1 데이터 삭제
            * `delete from t1 join t2 ..` → t1 데이터 삭제
            * `delete from t1 join t2 .. join t3 ..` → t1 데이터 삭제
* from 절
    * 삭제 작업에 필요한 테이블을 명시
    * 조인 조건을 사용하여 다중 테이블에 대해 삭제 수행 가능
<br>
