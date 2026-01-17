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
### Single Column Index
* 하나의 컬럼을 기준으로 생성된 인덱스
* 특정 컬럼에 대한 검색, 정렬, 범위 조건을 빠르게 처리 가능
### Composite Index
* 두 개 이상의 컬럼을 기준으로 생성된 인덱스
* 다중 컬럼 검색에 효율적
* 인덱스 순서가 매우 중요
    * 왼쪽 컬럼부터 순서대로 조건이 사용되어야 인덱스 활용에 효과적 → `leftmost prifix rule`
    * 인덱스의 앞쪽 컬럼이 조건에 있으면 최적화 가능
    * 앞선 컬럼이 조건에 없다면 인덱스 활용이 어려움
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
 create table t1 (c1 varchar(100), c2 text(100), index idx1 (c1(3)));
 create unique index idx2 on t1(c2(3));
 ```
 ### Functional Index
 ### Full-Text Index
 ### Spatial Index
 ### Adaptive Hash Index
 <br>

# FULL SCAN vs INDEX SCAN

# 옵티마이저와 INDEX 선택
# 복합 INDEX 설계 전략
# INDEX 생성 및 삭제
# INDEX가 사용되지 않는 경우
# INDEX의 비용 (Trade-Off)
# 실행 계획(EXPLAIN)
