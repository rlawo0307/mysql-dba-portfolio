# INDEX란?
* 테이블의 데이터를 빠르게 찾기 위한 자료구조
* 실제 데이터는 테이블에 저장되고 인덱스가 그 위치를 가리키고 있음
<br>

# INDEX 구조
* InnoDB 인덱스의 기본 구조 : `B+Tree`
### B-Tree
* Balanced Tree 자료구조
* 모든 노드에 `키 + 데이터(또는 데이터 포인터)` 저장
* 내부 노드에 실제 데이터가 존재함
* leaf 노드끼리 연결되어 있지 않음
* 탐색 시간 : `O(log N)`
* 원하는 키를 찾으면 중간 노드에서 바로 종료 가능
* 범위 검색에 비효율적
### B+Tree
* Balanced Tree 자료구조
* 내부 노드에 `키 + 포인터` 저장
* **실제 데이터는 leaf 노드에 저장되어 있음**
* leaf 노드끼리 좌우로 연결되어 있으며 정렬된 `linked list` 형태
* 탐색 시간 : `O(log N)`
* 범위 검색에 매우 유리
#### MySQL(InnoDB)은 왜 B+Tree를 선택했을까?
* DB 특성 상 범위 검색이 많음
* leaf 노드가 연결되어 있기 때문에 범위 검색 시 순차 접근 가능
* 내부 노드에 데이터가 없어 더 많은 키를 저장할 수 있음
    * 한 노드가 더 많은 자식 노드를 가질 수 있음 (fan-out 증가)
    * 트리 높이 감소
    * 디스크 페이지 접근 횟수 최소화 (디스크 I/O 감소)
 <br>

# INDEX 종류
* 구조 기준
    * B-Tree Index
    * Hash Index
* 물리적 저장 구조 기준
    * Clustered Index
    * Secondary Index
* 제약 / 기능 기준
    * Primary Key Index
    * Unique Index
    * Non Unique Index
* 컬럼 구성 기준
    * Single Column Index
    * Composite Index
    * Covering Index
* 컬럼 값 처리 기준
    * Prefix Index
    * Functional Index (MySQL 8.0+)
* 특수 목적
    * Full-Text Index
    * Spatial Index
* InnoDB 내부 자동 인덱스
    * Adaptive Hash Index
### Hash Index
* hash table 기반 인덱스
* 키 값을 hash function으로 변환해 저장
* 범위 검색 및 정렬 불가
* **InnoDB에서는 직접 생성 불가**
* 필요한 경우 `Adaptive Hash Index(AHI)`를 내부적으로 사용
### Clustered Index
* 테이블의 실제 데이터(row)가 pk 기준으로 정렬된 구조로 저장되는 인덱스
* 테이블 당 하나만 존재 가능
* 구조 : `B+Tree` 구조
* Secondary Index의 기준점 역할
* InnoDB에서는 `Primary Key Index` = `Clustered Index`
* pk 검색 성능 매우 빠름
* 범위 검색에 효율적
* clustered index 선택 우선순위
    1. primary key
    2. not null + unique 인덱스 중 첫 번째
    3. 내부적으로 생성한 row_id
### Secondary Index
* Clustered Index 가 아닌 모든 인덱스
* 실제 데이터가 아니라 pk 값을 저장하고 있는 인덱스
    * pk가 길면 secondary index의 크기 증가
    * 전체 성능 저하를 유발할 수 있음
* 테이블 당 여러 개 생성 가능
* 구조 : `B+Tree` 구조
* 동작 방식 (`Bookmark Lookup`/ `Back Lookup`)
    1. Secondary Index 탐색
    2. leaf 노드에서 pk 값 획득
    3. Clustered Index로 재탐색
    4. 실제 row 데이터 접근
### Primary Key Index
* `primary key`로 생성되는 인덱스
* InnoDB에서는 `Clustered Index`와 동일
* 테이블의 실제 row 데이터가 저장되는 유일한 인덱스
### Unique Index
* `unique` 제약으로 생성되는 인덱스
* 중복을 허용하지 않는 `Secondary Index`
### Non Unique Index
* `index` 또는 `key`로 생성되는 일반 인덱스
* 중복을 허용하는 `Secondary Index`
```sql
create table t1 (c1 int, c2 int, index idx1(c1));
create index idx2 on t1(c2);
drop index idx1 on t1;
drop index idx2 on t1;
```
### Single Column Index
* 하나의 컬럼을 기준으로 생성된 인덱스
* 특정 컬럼에 대한 검색, 정렬, 범위 조건을 빠르게 처리 가능
### Composite Index
* 두 개 이상의 컬럼을 기준으로 생성된 인덱스
* 다중 컬럼 검색에 효율적
* **컬럼 순서 결정 전략**
    * 선두 컬럼부터 조건에 사용되어야 인덱스 활용에 효과적
        * `Leftmost Prefix Rule`
    * `=` 조건이 걸린 컬럼을 앞에 두기
        * equality 조건은 인덱스 탐색 범위를 가장 많이 줄여줌
        * selectivity가 높은 컬럼일수록 효과적
    * `selectivity`가 높은 컬럼을 앞에 두기
        * selectivity = (서로 다른 값 수) / (전체 row 수)
        * selectivity가 높을 수록 필터링 효과가 큼
    * 범위 조건이 걸린 컬럼은 뒤에 두기
        * 범위 조건 이후 컬럼은 인덱스 효율 급감
    * `order by` 컬럼을 인덱스에 포함
        * 정렬 생략 가능 → 성능 이득
    * `group by` 컬럼을 인덱스 앞에 두기
        * group by 컬럼 순서가 인덱스와 일치해야 효과적
    * 가능하다면 `covering index` 고려
        * 쿼리에 필요한 컬럼을 인덱스에 모두 포함
        * back lookup을 제거하여 성능을 높일 수 있음
        * 자주 조회되는 컬럼을 뒤에 포함
```sql
create table t1 (c1 int, c2 int, c3 int, c4 int, index idx1(c1, c2));
create index idx2 on t1(c3, c4);
```
### Covering Index
* 쿼리에 필요한 모든 컬럼이 인덱스에 포함된 상태
    * 별도의 물리적인 인덱스 종류가 아님
    * 인덱스만으로 쿼리를 처리할 수 있는 **역할(특성)**을 의미
* 인덱스에 저장된 `key 값`이 곧 쿼리에 필요한 `데이터`가 되는 상태
    * 별도로 테이블을 읽지 않아도 됨 → `back lookup` 불필요
    * 디스크 I/O 감소 → 쿼리 성능 향상
 * 즉, **`Secondary index`가 쿼리에서 요구하는 컬럼 전부를 커버하고 있어서 `back lookup` 없이 인덱스만으로 쿼리 결과를 완전히 처리할 수 있는 상태**
### Prefix Index
 * 문자열 컬럼의 앞부분(접두사)만 인덱스로 만드는 방식
 * 문자열의 일부만 인덱싱해서 저장 공간 절약
 * `varchar`, `text`, `blob` 타입에 사용
    * 특히, `text` 또는 `blob`은 전체 컬럼에 대한 인덱싱을 허용하지 않음
* 인덱스 키 중복이 발생할 수 있음
    * `unique index` 생성으로 중복 방지는 가능하지만 주의 필요
        * 접두사 길이가 짧으면 다른 문자열이 같은 접두사로 인식될 수 있음
        * 결과적으로 완전한 중복 방지를 보장할 수 없음
    * 중복 방지가 목적이라면
        * 접두사 길이를 충분히 길게 설정
        * **가능하면 전체 컬럼 인덱싱이 안전**
 ```sql
 create table t1 (c1 varchar(100), c2 text, index idx1 (c1(3)));
 create unique index idx2 on t1(c2(3));
 ```
### Functional Index
 * 컬럼 값에 함수를 적용한 결과를 인덱스로 생성
 * 컬럼에 함수가 적용된 조건에서도 인덱슬 탈 수 있게 해줌
    * 일반 인덱스는 컬럼에 함수가 적용되면 인덱스를 사용 할 수 없음. 단, 값에 함수가 적용된 경우는 인덱스 사용 가능
        * ex) `lower(c1) = 'a'` → 인덱스 사용 불가
        * ex) `c1 = lower('A')` → 인덱스 사용 가능
* 인덱스 정의와 쿼리의 함수 표현식이 완전히 동일해야 사용 가능
* 대소문자 무시 검색, 날짜 가공 검색에서 유용
    * `lower()`, `upper()`, `date()`, `year()` 등을 자주 사용할 때
```sql
create table t1 (c1 char);
create index idx on t1((lower(c1)));
explain analyze select * from t1 where lower(c1) = 'a';
```
```sql
| -> Index lookup on t1 using idx (lower(c1)='a')  (cost=0.35 rows=1) (actual time=0.0113..0.0113 rows=0 loops=1)
 |
```
### Adaptive Hash Index
* InnoDB가 내부적으로 자동 생성하는 인덱스
* 자주 사용되는 B+Tree 페이지를 메모리 내부에 hash 구조로 캐싱
    * B+Tree 인덱스를 대체하지 않으며, 보조적인 캐시 역할 수행
* **사용자가 직접 인덱스를 생성/수정/삭제할 수 없음**
* equality(=) 검색에 매우 빠름
* 범위 검색, 정렬, like 검색에는 사용되지 않음
#### 현재 AHI 상태 확인
```sql
show variables like 'innodb_adaptive_hash_index';
```
#### AHI 활성화/비활성화
* `global` 설정이므로 권한 필요
    * `SUPER` : 서버 전역 제어 권한
    * `SYSTEM_VARIABLES_ADMIN` : 시스템 변수 변경 권한
* 새 커넥션부터 적용 (기존 세션에는 영향 없음)
```sql
set global innodb_adaptive_hash_index = ON;
set global innodb_adaptive_hash_index = OFF;
```
 <br>

# FUll TABLE SCAN vs INDEX SCAN
* [→ 각 scan 방식에 대한 실행 계획 자세히 보기 (기본)](labs/explain-table-access-types.md)
* [→ 각 scan 방식에 대한 실행 계획 자세히 보기 (심화)](labs/explain-advanced-index-scan.md)

## Full Table Scan
* 테이블의 모든 row를 처음부터 끝까지 순차적으로 읽는 방식
* 조건과 상관없이 전체 데이터에 접근
* 데이터 양이 많을수록 비용 급증
* 사용되는 경우
    * 조건이 없을 때
    * 인덱스가 없거나 인덱스를 사용할 수 없을 때
    * 인덱스 접근 비용보다 전체 스캔 비용이 싸다고 판단할 때
## Index Scan
* 인덱스를 통해 필요한 row만 빠르게 탐색하는 방식
* B+Tree 구조를 이용한 `O(log N)` 탐색
* 종류
    * 기본 스캔 방식
        * Index Lookup (Unique Scan)
        * Index Range Scan
        * Index Full Scan
    * 스캔 최적화 전략
        * Index Skip Scan
        * Loose Index Scan
        * Backward Index Scan
    * 다중 인덱스 활용
        * Index Merge
### Index Lookup
* 인덱스의 특정 키 값을 정확히 찾는 방식
    * 보통 `=` 조건을 사용할 때 발생
    * ex) `select * from t1 where c1 = 100;`
```text
* 동작 방식
    1. Secondary Index 탐색
    2. 조건 키로 B+Tree 탐색
    3. leaf 노드에서 pk 획득
    4. Clustered Index 재탐색 (back lookup)
    5. 1건의 row에 접근
```
* 가장 빠른 인덱스 접근 방식
* 접근하는 인덱스 페이지 수가 매우 적음
### Index Range Scan
* 특정 범위의 인덱스 키를 연속으로 읽는 방식
    * `범위` 조건을 사용할 때 발생
    * ex) `select * from t1 where c1 between 10 and 20;`
```text
* 동작 방식
    1. Secondary Index 탐색
    2. 조건 시작 지점 탐색
    3. leaf 노드 범위 스캔
    4. 다수의 pk 획득
    5. Clustered Index 재탐색 (back lookup)
    6. 여러 건의 row에 접근
```
* leaf 노드가 연결되어 있어 디스크 접근에 효율적
* 결과 row 수가 많아질수록 성능 저하가 발생할 수 있음
* back lookup 횟수가 성능 병목이 되기 쉬움
### Index Full Scan
* 인덱스 전체를 처음부터 끝까지 읽는 방식
    * 인덱스는 사용하지만 조건 필터링 효과가 거의 없음
    * ex) `select c1 from t1;` (c1에 covering index가 있는 경우)
    * ex) `select * from t1 order by c1;`
```text
* 동작 방식
    1. Secondary Index 탐색
    2. leaf 노드 전체 순차 스캔
    3. 모든 pk 획득
    4. Clustered Index 재탐색 (back lookup)
    5. 전체 row에 접근
```
* 사용되는 이유
    * 인덱스가 테이블보다 훨씬 작기 때문에
        * 조건 필터링이 없어도 읽는 양이 훨씬 작음 → 디스크 I/O 감소
    * order by 수행 시
        * index는 이미 정렬된 상태 → 성능 이득
    * 캐시 적중 가능성이 높을 때
        * 인덱스는 작고 자주 사용됨
        * 버퍼 풀에 존재할 가능성이 높음
        * 디스크 I/O 없이 메모리에서 처리
    * covering index일 경우 table full scan보다 압도적으로 빠름 
### Index Skip Scan
* `복합 인덱스`에서 먼저 나온 컬럼이 조건에 없어도 옵티마이저가 선두 컬럼의 가능한 값들을 `건너뛰며` 내부적으로 여러 번 `Index Range scan`을 수행
* 옵티마이저가 인덱스 통계를 통해 선두 컬럼의 서로 다른 값 개수(distinct)를 파악
* 사용되는 경우
    * 선두 컬럼의 distinct 값이 적을 때 사용
    * 후행 컬럼의 조건 선택도가 높을 경우 사용
    * 테이블 크기가 크고 full scan이 부담되는 경우
### Loose Index Scan
* 인덱스에서 필요한 값만 loose하게 읽어 불필요한 row 접근을 건너뛰는 방식
* 사용되는 경우
    * 각 그룹마다 첫 번째 값만 필요한 경우
    * ex) `group by + min()/max()`, `distinct`, 집계 쿼리 등
    * 단, group by 컬럼이 인덱스 순서와 동일해야 하며 집계 대상 컬림이 다음 순서에 나와야 함
### Backward Index Scan
* 인덱스를 정렬 반대 방향으로 탐색하는 방식
* 보통 B+Tree 인덱스는 오름차순으로 정렬되어 있음
    * 마지막 leaf 노드부터 탐색하여 역순으로 읽어서 반환
    * B+Tree는 양방향으로 leaf 노드가 연결되어 있기 때문에 가능
* 사용되는 경우
    * 정렬 기준 컬럼에 인덱스가 있으며 내림차순으로 정렬할 때
    * ex) `order by c1 desc;`
* order by 컬럼이 인덱스 순서와 다를 경우 사용 불가
* 혼합 정렬일 경우 사용 불가
    * ex) `order by c1 desc, c2 asc;`
### Index Merge
* 하나의 인덱스로 처리할 수 없는 조건을 여러 개의 인덱스를 동시에 사용해서 해결하는 방식
```text
* 동작 방식
    1. idx1로 조건에 맞는 pk 목록 수집
    2. idx2로 조건에 맞는 pk 목록 수집
    3. 두 결과를 집합 연산
    4. 최종 pk 목록으로 row 접근
```
* 종류
    * Index Merge Intersection
        * ex) `where a = 10 and b = 20;`
        * 두 인덱스 결과의 교집합
    * Index Merge Union
        * ex) `where a = 10 or b = 20;`
        * 두 인덱스 결과의 합집합
        * 중복 제거 필요
    * Index Merge Sort-Union
        * ex) `where a < 10 or b < 20;`
        * range scan 결과를 정렬 후 merge
        * 비용이 커서 잘 안 쓰임
* 장점
    * 복합 인덱스가 없어도 다중 조건 처리 가능
    * 기존 단일 인덱스를 재활용 가능
* 단점
    * Secondary index를 여러 개 스캔 해야 함
    * pk 목록을 병합하는데 비용 발생
    * 메모리 사용량 증가
    * 복합 인덱스보다 거의 항상 느림
* **옵티마이저가 `Index Merge`를 사용하였다면 인덱스 설계가 부족하다는 신호!**
    * `Index Merge`는 옵티마이저의 임사방편이며
    * 가능하다면 복합 인덱스로 대체하는 것이 좋음
<br>

# OPTIMIZER와 SCAN 방식
### Optimizer란?
* SQL을 어떻게 실행할지 결정하는 MySQL의 핵심 컴포넌트
    * InnoDB는 `Cost-Based-Optimizer(CBO)` 사용
* 여러 실행 계획 후보 중 가장 cost가 낮다고 판단되는 계획을 선택
* scan 방식 외에 join 순서, join 방식, 조건 평가 순서 등 쿼리 실행 전체 전략을 결정
* SQL 실행 결과는 항상 동일하며 성능에 차이가 있음
* 옵티마이저의 비용 계산은 **통계 정보를 기반으로 한 추정값**임
* 옵티마이저의 비용 계산 기준
    * 예상 읽기 row 수
        * selectivity (선택도)
        * cardinality (카디널리티)
    * I/O 비용
        * 디스크 페이지 접근 횟수
        * 테이블 페이지 vs 인덱스 페이지
    * CPU 비용
        * 조건 비교, 정렬, 집계 연산 등
    * Back Lookup 비용
### 그렇다면 Index Scan이 항상 빠를까?
* index scan이 항상 table full scan보다 빠른 것은 아님
* 옵티마이저는 table scan과 index scan을 모두 후보로 두고 cost을 비교
* **index scan을 선택하지 않는 경우**
    * 조건에 인덱스 컬럼이 사용되지 않은 경우
    * 조건을 인덱스로 사용할 수 없는 형태로 작성한 경우
        * 함수 사용
        * 암시적 형 변환
        * 산술 연산
        * `like '%xxx'` 패턴
        * 부정 조건 (`not`, `!=`, `<>` 등)
    * 복합 인덱스 규칙을 위반한 경우
        * 선두 컬럼이 조건에 사용되지 않은 경우
        * 범위 조건 이후 컬럼을 사용한 경우
    * index scan이 비용 관점에서 불리한 경우
        * 조건의 selectivity가 낮아서 전체 row의 많은 비율을 읽어야 하는 경우
        * 결과 row가 많아서 back lookup 비용이 커진 경우
    * 통계 정보가 부정확할 경우
    * 옵티마이저 힌트 사용으로 스캔 방식을 강제한 경우
<br>

# INDEX의 비용 (Trade-Off)
* 인덱스는 조회 성능을 크게 향상시키지만, 항상 좋은 것은 아님
    * 저장 공간 비용 발생
        * 인덱스는 테이블 데이터와 별도로 저장됨
        * 인덱스가 많을수록 디스크에 공간이 더 많이 필요
    * 쓰기 작업 비용 증가
        * `insert`, `update`, `delete` 시 인덱스도 함께 갱심되어야 함
        * 인덱스가 많을수록 쓰기 작업 비용과 지연 시간 증가
    * 복잡성 증가로 인한 성능 저하
        * 인덱스 설계가 잘못되면 쿼리 최적화에 악영향을 끼칠 수 있음
        * 불필요한 인덱스가 많으면 옵티마이저가 잘못된 실행 계획을 선택할 수 있음
        * 인덱스가 많을수록 옵티마이저가 실행 계획을 결정하는데 더 많은 계산 필요
### **★ 중요) 그렇다면 인덱스 설계 시 고려해야 할 것은 무엇인가? ★**
```text
1. 자주 조회하는 컬럼에만 인덱스 생성하기
2. 복합 인덱스는 필요한 컬럼만 최소한으로 포함하기
3. 쓰기 작업과 조회 작업의 균형을 고려하기
4. 주기적인 인덱스 모니터링과 튜닝 필요
```
<br>