# Partitioning이란?
* 하나의 큰 테이블을 여러 개의 작은 단위(partition)로 나누어 저장하는 기술
    * partition key를 기준으로 partition을 나눔
* 논리적으로는 하나의 테이블, 물리적으로는 여러 partition으로 분리
* 사용자 입장에서는 하나의 테이블처럼 조회/조작 가능
* 사용하는 이유
    * partition pruning을 통해 조회 성능을 향상시킬 수 있음
    * 데이터 관리 효율성
        * partition 단위로 데이터 관리 가능
        * 특정 partition만 빠르게 삭제/관리 가능
    * 대용량 데이터 처리
        * 데이터를 여러 partition으로 분산 저장
        * I/O 분산 및 관리 용이
* 주의할 점
    * partitioning이 향상 성능을 향상시키는 것은 아님
    * partition key 조건이 없으면 결국 모든 partition을 조회하게 됨
<br><br>

# Partition 종류
* range partition
* range columns partition
* list partition
* hash partition
* key partition
### range partition
* 특정 값의 범위를 기준으로 partition을 나누는 방식
* 컬럼에 expression 사용 가능
* 복합 컬럼을 직접 사용하는 것은 불가능
    * expression을 통해 간접적으로 결합하여 사용
* 범위 조건에 최적화되어 있음
* partition pruning 효과가 가장 잘 나타남
```sql
create table t1(c1 int, c2 int) partition by range(c1)(
    partition p1 values less than (10), -- c1 < 10 → p1
    partition p2 values less than (20), -- c1 < 20 → p2
    partition p3 values less than maxvalue -- 그 이상 → p3
);
```
### range columns partition
* 특정 값의 범위를 기준으로 partition을 나누는 방식
* range partition과 달리 **컬럼에 expression을 사용할 수 없음**
* 컬럼 값 자체를 기준으로 범위를 비교
* 날짜/문자열/복합 컬럼에 대해 직관적인 partition 설계 가능
* 범위 조건과 직접 비교되므로 partition pruning이 안정적으로 동작
```sql
create table t1(c1 date, c2 int) partition by range columns(c1)(
    partition p1 values less than ('2025-01-01'), -- c1 < '2025-01-01' → p1
    partition p2 values less than ('2026-01-01'), -- c1 < '2026-01-01' → p2
    partition p3 values less than (maxvalue)      -- 그 이상 → p3
);
```
```sql
-- 복합 컬럼 사용 예시
create table t1(c1 int, c2 int) partition by range columns(c1, c2)(
    partition p1 values less than (10, 100),
    partition p2 values less than (20, 200),
    partition p3 values less than (maxvalue, maxvalue)
);
```
### list partition
* 특정 값의 목록을 기준으로 partition을 나누는 방식
* 명시적인 값을 매핑
* default partition 개념이 없음
    * 정의되지 않은 값이 들어오면 오류 발생
    * 나머지가 있는 경우 range partition을 사용하는 것을 추천
* 값의 종류가 제한적일 경우 유용
```sql
create table t1(c1 int, c2 int) partition by list(c1)(
    partition p1 values in (1, 2, 3),
    partition p2 values in (4, 5, 6)
);
```
### hash partition
* 사용자가 정의한 hash key를 이용해 데이터를 균등하게 분산하는 방식
    * 단순 연산, 복합 계산, 함수 사용 모두 가능
    * 사용자 정의 함수(UDF) 사용 불가
    * 비결정적 함수 사용 불가
    * 복합 컬럼을 직접 지정하는 것은 불가
        * expression을 통해 간접적으로 결합하여 사용
* 실제 partition 결정은 MySQL 내부 hash 함수가 수행
    * key 결과를 기반으로 MySQL 내부 hash 함수가 partition을 결정
    * 데이터가 어떤 partition에 들어갈지 예측할 수 없음
        * partition pruning 효과가 제한적
* 각 partition 정의 없이 partition 개수만 지정
    * 내부적으로 hash 값을 계산한 후 partition 개수로 분배
* 특정 범위 조건 없이 전체 데이터가 고르게 사용되는 경우 사용
* I/O 분산이 필요한 경우 사용
```sql
create table t1(c1 int, c2 int) partition by hash(c1) partitions 4; -- hash key : c1
create table t1(c1 int, c2 int) partition by hash(c1 + 100) partitions 4; -- hash key : c1 + 100
                                                                          -- 간접적으로 hash 제어 가능
create table t1(c1 int, c2 int) partition by hash(c1 + c2) partitions 4; -- 복합 연산 사용 가능
```
### key partition
* MySQL의 내부 hash 함수를 이용해 데이터를 균등하게 분산하는 방식
    * 실제 알고리즘은 공개되지 않음
    * 복합 컬럼 사용 가능
* 각 partition 정의 없이 partition 개수만 지정
* hash partition보다 더 균등한 분산 가능
* 사용은 간편하나 직접적인 동작 제어가 어려움
* 데이터가 어떤 partition에 들어갈지 예측할 수 없음
    * partition pruning 효과가 제한적
```sql
create table t1(c1 int, c2 int) partition by key(c1) partitions 4; -- 각 partition 정의 없이 개수만 지정
create table t1(c1 int, c2 int) partition by key(c1, c2) partitions 4; -- 복합 컬럼 사용 가능
```
<br>

# Partition Key
* partition을 나누는 기준이 되는 컬럼 또는 expression
* partition 성능은 key 설계에 의해 크게 좌우됨
* 특히, partition pruning과 직접적으로 연결됨
* 설계 원리
    * 데이터 분할 기준이 아닌 **조회 패턴**을 기준으로 선택해야 함
    * pruning이 잘 동작하도록 설계하는 것이 핵심
    * range columns 방식이 가장 직관적이고 안정적
### 좋은 partition key 설계
    - where 조건에 자주 사용되는 컬럼
    - 범위 조건으로 자주 조회되는 컬럼
    - 데이터가 균등하게 분포되는 컬럼
    - where 조건에서 가공 없이 사용되는 컬럼 또는 expression
### 잘못된 partition key 설계
    - where 조건에 사용되지 않는 컬럼
    - equality 조건 위주의 컬럼
    - 함수/연산이 필요한 경우
    - partition 개수를 과도하게 증가시키는 설계
<br>

# Partition Pruning
* 파티셔닝 환경에서 쿼리 조건을 기반으로 필요한 partition만 읽는 최적화 기법
* 대용량 테이블에서 성능에 가장 큰 영향을 미치는 요소
    * 항상 성능을 향상시키는 것은 아님
    * partition 수가 적으면 효과가 제한적
    * partition이 너무 많으면 오히려 관리 비용 증가
    * pruning이 실패하면 오히려 full scan보다 느려질 수 있음
* explain의 `partitions` 컬럼으로 pruning 여부 확인 가능
    * 조회된 partition만 표시됨
    * 전체 partition이 표시되면 pruning 실패
* 원리
    * MySQL 옵티마이저는 쿼리 실행 전에 어떤 partition이 필요한지 판단
    * `partition key`와 where 조건을 비교하여 읽을 partition을 결정
    * 불필요한 partition은 접근하지 않음
* 핵심 조건
    * where 조건이 **partition key**를 기준으로 변환 가능해야 함
        * partition key와 동일하거나
        * 해당 표현식으로 환산 가능해야 함
    * 조건이 상수 기반으로 실행 전에 결정되어야 함
        * 옵티마이저가 실행 계획 단계에서 접근할 partition을 확정할 수 있어야 함
### pruning 성공 및 실패 상황
```sql
create table c1 (c1 date, c2 int) partition by range(year(c1))(
    partition p1 values less than (2025),
    partition p2 values less than (2026),
    partition p3 values less than (2027)
);
```
* pruning 성공 조건
    * partition key와 비교 가능한 상수 조건
    * 컬럼에 함수가 적용되었지만 partition key와 동일한 경우
```sql
select * from t1 where year(c1) = 2025;
-- 상수 비교
-- where 조건이 partition key와 동일
-- p2 조회

select * from t1 where c1 = '2025-05-01';
-- 상수 비교
-- partition key가 year(c1)이므로 MySQL이 내부적으로 year()을 적용해서 계산
-- year('2025-05-01') = 2025로 계산하여 판단
-- p2 조회

select * from t1 where c1 between '2025-01-01' and '2026-01-01';
-- 상수 비교
-- partition key가 year(c1)이므로
-- 각각 year('2025-01-01') = 2025, year('2026-01-01') = 2026으로 계산
-- 결과적으로 where year(c1) between 2025 and 2026으로 판단
-- between은 양끝 포함이므로 '2026-01-01'도 포함됨
-- p2, p3 조회
```
* pruning 실패 조건
    * partition key가 where 조건에 사용되지 않은 경우
    * 컬럼에 연산이 적용되어있는 경우
    * 컬럼에 함수가 적용되었고 partition key와 동일하지 않은 경우
    * bind parameter를 사용한 경우
    * or 조건 사용
```sql
select * from t1 where c2 = 10;
-- partition key와 다른 컬럼 조건 사용

select * from t1 where c1 + interval 1 day = '2025-05-24';
-- 컬럼에 연산 적용

select * from t1 where month(c1) = 5;
-- 컬럼에 함수 적용. partition key와 다름

select * from t1 where c1 = ?
-- bind parameter 사용
-- 실행 시점까지 값이 확정되지 않음
-- optimizer가 partition을 결정할 수 없음

select * from t1 where c1 = '2025-05-01' or c2 = 10;
-- or 조건 사용
-- optimizer가 partition을 단순하게 확정하기 어려움
-- pruning이 제한되거나 실패할 수 있음
```
<br>