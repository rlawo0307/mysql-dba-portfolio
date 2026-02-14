# JOIN이란?
* 두 개 이상의 테이블을 조건을 기준으로 결합하여 하나의 결과 집합을 만드는 연산
* JOIN = `Cartesian Product + Filter`
    * 두 테이블의 카테시안 곱으로 모든 조합을 생성한 후 조건 필터링
    * 논리적 모델 기준 설명
    * 실제 엔진은 최적화된 알고리즘 사용
<br>

# 옵티마이저와 JOIN
* 조인 과정에서의 기본적인 의사 결정은 옵티마이저에 의해 결정됨
* 사용자가 작성한 쿼리의 테이블 순서대로 조인이 수행되지 않을 수 있음
    * 옵티마이저가 내부적으로 조인 순서를 변경하여 더 효율적인 실행 계획을 선택할 수 있음
### 결정 요소
* 조인 테이블 순서
    * 옵티마이저는 어떤 테이블을 먼저 읽을지 결정
        * 먼저 읽는 테이블 = `driving table`
        * 반복 탐색되는 테이블 = `driven table`
    * driving table 선택 기준
        * cardinality (낮을수록 유리)
        * selectivity (높을수록 유리)
        * 인덱스 존재 여부
        * 조인 조건의 효율성
* 조인 조건 평가 순서
    * selectivity가 높은 조건을 먼저 적용하면 불필요한 row 탐색을 줄일 수 있음
* 인덱스 사용 여부
    * 조인 성능은 인덱스 사용 여부에 큰 영향을 받음
    * 특히, driven table의 조인 컬럼에 인덱스가 존재하면 성능이 크게 향상됨
* 조인 알고리즘 선택
### 옵티마이저 판단 흐름
```text
1. 가능한 모든 조인 순서 조합 생성 (※휴리스틱 기반으로 탐색 범위 축소)
2. 각 조합의 예상 cost 계산
3. 가장 cost가 낮은 최적의 조합 선택
```
* join 성능 최적화 핵심
    * 조인 컬럼에 인덱스 생성
    * cardinality가 높은 테이블이 driving table이 되도록 유도
    * selectivity가 높은 조건에 사용되는 테이블이 driving table이 되도록 유도
    * where로 먼저 필터링하기
        * inner join의 경우, on과 where 조건이 동일하게 최적화 될 수 있음
        * outer join의 경우, 결과에 영향을 줄 수 있음
            * on 조건은 조인 과정에 적용
            * where 조건은 조인 결과에 적용
    * 데이터 타입 일치시키기
    * 불필요한 outer join 제거
    * 불필요한 컬럼 조회 피하기 (select * 등)
    * 정확한 통계 정보 유지
<br>

# JOIN 종류
### INNER JOIN
* 두 테이블의 교집합
* 두 테이블에서 조건을 만족하는 행만 반환
* on과 where 조건이 논리적으로 동일하게 최적화 될 수 있음
    * 옵티마이저가 조건 순서를 바꿀 수 있음
    * 성능 차이는 거의 없음
* 가장 많이 사용되는 조인 방식
```sql
select * from t1 inner join t2 on t1.c1 = t2.c1;
```
### LEFT OUTER JOIN
* 왼쪽 테이블의 모든 행을 유지하고 오른쪽 테이블은 조건이 맞을때만 결합
    * 논리적으로 왼쪽 테이블의 결과를 유지
    * 물리적 실행에서는 옵티마이저가 조인 순서를 변경할 수 있음
    * 즉, **왼쪽 테이블이 driving table이 아닐 수 있음**
* 매칭되지 않는 경우 오른쪽 컬럼은 null
* 조인의 방향성이 결과에 영향을 주기 떄문에 조인 테이블 순서 변경이 자유롭지 않음
    * 옵티마이저가 조인 순서 변경을 아예 안한다는 것은 아님
    * 조인 순서 최적화를 위해 불필요하다면 inner join으로 변경하는 것이 좋음
* 조건 위치에 따라 결과가 달라질 수 있음
    * 옵티마이저는 on 조건과 where 조건 순서를 함부로 변경하지 않음
    * 사용자 역시 조건의 위치를 신중히 선택해야 함
        * 특히, where 조건은 null row를 제거할 수 있으므로 inner join처럼 동작할 수 있으니 주의 필요
```sql
select * from t1 left join c2 on t1.c1 = t2.c1;
select * from t1 right join c2 on t1.c1 = t2.c1; -- right outer join은 방향만 반대
```
### CROSS JOIN
* 조인 조건 없이 모든 조합 생성
* 순서 cartesian product
```sql
select * from t1 cross join t2;
```
### SELF JOIN
* 자기 자신과의 조인
* alias 필수
* 계층 구조에 많이 사용됨
```sql
select * from t1 t join t1 s on t.c1 = s.c2;
```
## 기타 JOIN
* SQL에 직접적인 키워드는 없지만 옵티마이저가 내부적으로 변환하여 사용할 수 있는 조인
### SEMI JOIN
* 두 테이블 중 하나에서 매칭되는 row가 존재하는지만 확인하는 조인
* 매칭이 하나라도 있다면 true
    * 몇 개가 매칭되는지는 중요하지 않음
* 중복 row를 생성하지 않음
* driven table을 모두 탐색하지 않을 수 있음
    * 매칭 발견 시, 즉시 종료
* 변환되는 경우
    * exists
    * in
```sql
select * from t1 where exists(select 1 from t2 where t2.c1 = t1.c1);
select * from t1 where c1 in (select c1 from t2);
```
### ANTI JOIN
* 매칭 row가 없는 경우만 반환하는 조인
* driven table에서 매칭이 발견되는 해당 row는 즉시 제외됨
* 변환되는 경우
    * not exists
    * left join + is null
```sql
select * from t1 where not exists(select 1 from t2 where t2.c1 = t1.c1);
-- not exists는 null 영향이 없으므로 안전하게 변환 가능

select * from t1 left join t2 on t1.c1 = t2.c1 where t2.c1 is null;
-- on 조건에서 매칭되지 않는 경우, t2의 컬럼이 null로 채워진 row가 생성됨
-- 그 후 is null 조건에 의해 매칭되지 않는 row 반환
```
#### +) `not in`은 경우에 따라 anti join으로 변환되지 않을 수도 있음
```sql
select * from t1 where c1 not in (select c1 from t2);
```
* `t1.c1 = {10}`, `t2.c1 = {1, 2, null}`일 경우
    * 논리적으로, 10은 {1, 2, null}에 없으므로 반환될 것을 예상
    * not in 조건은 `10 != 1 && 10 != 2 && 10 != null`로 변환되어 계산
    * 이때, `10 != null`은 unknown이므로 where 조건은 항상 false
    * 결과적으로 10은 반환되지 않음
    * anti join으로 변환 불가
* anti join으로 변환될 수 있는 경우
    * 서브쿼리 컬럼이 not null일 때
    * 옵티마이저가 null이 없다고 확신할 때
    * 내부적으로 null 제거가 가능할 때
<br>

# JOIN 알고리즘
* 두 테이블을 어떤 방식으로 결합할 것인지를 결정하는 실행 전략
### Nested Loop join (NLJ)
```sql
for each row in driving table:
    find matching rows in driven table
```
* OLTP 환경에서 가장 많이 사용되는 조인 방식
* 높은 selectivity와 인덱스가 존재할 경우 매우 효율적
    * 특히, driven table에 인덱스가 있는 경우 성능이 향상될 수 있음
* 단점 : random I/O 발생
    * 인덱스가 있어도 대량 데이터 조인에서는 random I/O가 여전히 느림
### Block Nested Loop (BNL)
* NLJ의 개선 버전
* 목표 : `driven table 접근 횟수를 줄여 random I/O를 줄이자`
* driving table을 한 row씩 읽는 대신, 여러 row를 버퍼에 저장
* driven table을 scan하면서 버퍼와 비교
* 단점
    * 여전히 인덱스를 활용하지 못함
    * 여전히 full scan 필요
    * 메모리 사용량 증가
* 인덱스를 사용할 수 없는 경우의 fallback 전략
* hash join 도입 이후 사용성이 감소됨
### Batched Key Access (BKA)
* BNL + 인덱스 최적화
* 목표 : `인덱스를 사용하면서 random I/O를 줄이자`
* 여러 인덱스 키를 모아서 정렬 후 한 번에 인덱스에 순차적으로 접근
    * random I/O 감소
    * sequential I/O 증가
* 단점
    * 인덱스가 존재할 때만 사용 가능
    * optimizer switch가 활성화 되어 있어야 함
### Hash Join
```text
1. 작은 테이블을 사용하여 hash table 생성
2. 큰 테이블을 scan
3. 큰 테이블의 row를 key로 하여 hash table 탐색
```
* 장점
    * 대용량 조인에 매우 빠름
    * 인덱스 필요 없음
    * 분석 쿼리에 유리
* 단점
    * equi join(`=`)일 때만 사용 가능
    * 메모리 사용량 증가
## 알고리즘 선택 흐름
```text
작은 driving table + index → NLJ
대량 데이터 + index → BKA
index X → BNL 또는 Hash Join
대용량 데이터 + 낮은 selectivity → Hash Join
```
<br>