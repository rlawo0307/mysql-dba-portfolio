```
[참고]

view : select를 저장해둔 가상 테이블
stored procedure : 여러 SQL을 묶어 둔 호출 가능한 프로그램
trigger : 특정 DML 이벤트 발생 시 자동 시행되는 이벤트 기반 로직
```
<br>

# View
* 하나의 select 문을 이름 붙여 저장한 객체
* 원본 테이블을 바탕으로 결과를 보여주는 가상 테이블
* 실제 데이터를 저장하고 있지 않음 
* 쿼리 결과를 저장하지 않기 때문에 매번 다시 실행됨
* 장점
    * 복잡한 조회를 단순화할 수 있음
    * 노출 범위를 제한하여 보안 강화
    * 논리적 독립성 보장
        * 애플리케이션은 뷰를 기준으로 조회
        * 실제 테이블 구조 변경은 내부에서 수행
* 단점
    * 항상 성능이 향상되는 것은 아님
    * 복잡한 뷰는 실행 계획이 난해해질 수 있음
    * 원본 테이블 구조 변경 시 영향을 받음
## view 처리 알고리즘
* 뷰가 포함된 쿼리 실행 시, 뷰 정의를 쿼리를 포함시켜서 전체 쿼리를 다시 작성함 → `Query Rewrite`
* 어떻게 뷰 정의를 포함시킬까에 대한 알고리즘
    * `merge` 방식
        * 대부분의 단순한 뷰를 처리할 때 사용
        * 뷰를 inline으로 삽입하여 쿼리를 합쳐버리는 방식
        * 매우 빠름
        * 원본 테이블의 인덱스 사용 가능
        * predicate pushdown 가능
        * 옵티마이저가 완전히 통제 가능
    * `temptable` 방식
        * merge 방식을 사용할 수 없는 경우 사용
            * 그룹 함수, 집계 함수, 윈도우 함수 등이 사용된 경우
            * 정렬이 필요한 경우
            * union, limit 등이 사용된 경우
            * 서브쿼리 구조인 경우
        * 뷰를 먼저 실행하여 임시 테이블 생성
        * 결과를 물리적으로 만든 후 이후 필터 적용
        * 원본 테이블의 인덱스 활용이 제한적
        * 원본 테이블에 대한 추가적인 predicate pushdown 불가능
        * 성능이 저하될 수 있음
* explain을 통해 사용된 알고리즘 확인 가능
    * 뷰가 아닌 실제 테이블이 직접 등장 → merge 방식
    * derived, temporary, Using temporary 키워드가 등장 → temptable 방식
## 동작 방식
```text
1. 파싱 단계에서 뷰가 존재하는지 확인
2. 뷰 정의 가져오기
3. Query Rewrite
4. 옵티마이저 최적화
5. 실행 (원본 테이블 또는 임시 테이블에 접근)
```
## 문법
```sql
create table t1(c1 int, c2 int); -- 원본 테이블

-- 뷰 생성
create view v as select c1 from t1 where c2 = 10;

-- 조회
select * from v;

-- 뷰 재정의
create or replace view v as select c2 from t1 where c1 = 1;

-- 데이터 수정
update v set c2 = c2 * 10 where c1 = 1; -- 모든 뷰가 수정 가능한 것은 아님
                                        -- updatable view 참고
-- 삭제
drop view v;
```
## Updatable View
* 뷰를 통해 데이터 udpate 시 원본 테이블의 데이터가 수정됨
* 모든 뷰가 수정가능한 것은 아님
* MySQL은 뷰 생성 시 `updatability flag`를 설정
    * `information_schema.views.is_updatable` 컬럼으로 확인 가능
* 뷰 정의에 join, 그룹 함수, 집계 함수, union, distinct 등이 포함된 경우 수정 불가능
## 옵션
* `with check option`
    * 뷰의 조건을 벗어나는 데이터가 삽입/수정되는 것을 방지
```sql
create view v as select * from t1 where c1 = 1 with check option; -- check 조건 : c1 = 1
```
* `sql security definer` (default)
    * 뷰 생성 시 설정 가능
    * 뷰를 **생성한** 사용자 권한으로 실행
    * 뷰 실행 시, 원본 테이블에 대한 권한 필요 없음
    * 뷰에 대한 접근 권한이 있으면 실행 가능
```sql
create view v sql security definer as select * from t1;
```
* `sql security invoker`
    * 뷰 생성 시 설정 가능
    * 뷰를 **실행하는** 사용자 권한으로 실행
    * 뷰 실행 시, 원본 테이블에 대한 권한 필요
    * 뷰 접근 권한 + 원본 테이블 접근 권한 모두 필요
```sql
create view v sql security invoker as select * from t1;
```