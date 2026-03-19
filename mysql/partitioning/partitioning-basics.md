# partitioning이란?
* 하나의 큰 테이블을 여러 개의 작은 단위(partition)로 나누어 저장하는 기술
* 논리적으로는 하나의 테이블
* 물리적으로는 여러 partition으로 분리
* 사용자 입장에서는 하나의 테이블처럼 조회/조작 가능
* 사용하는 이유
    * partition pruning을 통해 조회 성능을 향상시킬 수 있음 [→ partition pruning에 대한 내용 자세히 보기]()
        * 전체 데이터를 scan하지 않고 조건에 맞는 일부 partition만 조회
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
### range partition
* 특정 값의 범위를 기준으로 partition을 나누는 방식
* 범위 조건에 최적화되어 있음
* partition pruning 효과가 가장 잘 나타남 [→ partition pruning에 대한 내용 자세히 보기]()
```sql
create table t1(c1 int, c2 int) partition by range(c1)(
    partition p1 values less than (10), -- c1 < 10 → p1
    partition p2 values less than (20), -- c1 < 20 → p2
    partition p3 values less than maxvalue -- 그 이상 → p3
);
```
### list partition
* 특정 값의 목록을 기준으로 partition을 나누는 방식
* 명시적인 값을 매핑
* `나머지`를 처리하는 문법/로직 없음
    * 나머지가 있는 경우 range partition을 사용하는 것을 추천
* 값의 종류가 제한적일 경우 유용
```sql
create table t1(c1 int, c2 int) partition by list(c1)(
    partition p1 values in (1, 2, 3),
    partition p2 values in (4, 5, 6)
);
```
### hash partition
* 사용자가 정의한 hash expression을 이용해 데이터를 균등하게 분산하는 방식
    * 단순 연산, 복합 계산, 함수 사용 모두 가능
    * 사용자 정의 함수(UDF) 사용 불가
    * 비결정적 함수 사용 불가
    * 복합 컬럼을 직접 지정하는 것은 불가
        * expression을 통해 간접적으로 결합하여 사용
* hash expression은 사용자가 정의
* 실제 partition 결정은 MySQL 내부 hash 함수가 수행
    * expression 결과를 기반으로 MySQL 내부 hash 함수가 partition을 결정
    * 데이터가 어떤 partition에 들어갈지 예측할 수 없음
        * partition pruning 효과가 제한적
* 각 partition 정의 없이 partition 개수만 지정
    * 내부적으로 hash 값을 계산한 후 partition 개수로 분배
* 특정 범위 조건 없이 전체 데이터가 고르게 사용되는 경우 사용
* I/O 분산이 필요한 경우 사용
```sql
create table t1(c1 int, c2 int) partition by hash(c1) partitions 4; -- hash expression : c1
create table t1(c1 int, c2 int) partition by hash(c1 + 100) partitions 4; -- hash expression : c1 + 100
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


