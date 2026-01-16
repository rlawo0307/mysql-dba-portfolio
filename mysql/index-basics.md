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
## Hash Index
* hash table 기반 인덱스
* 키 값을 hash function으로 변환해 저장
* 범위 검색 및 정렬 불가
* **InnoDB에서는 직접 생성 불가**
* 필요한 경우 `Adaptive Hash Index(AHI)`를 내부적으로 사용
## Clustered Index
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
## Secondary Index
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
## Primary Key Index
## Unique Index
## Non Unique Index
## Single Column Index
## Composite Index
## Covering Index
# FULL SCAN vs INDEX SCAN
# 옵티마이저와 INDEX 선택
# 복합 INDEX 설계 전략
# INDEX 생성 및 삭제
# INDEX가 사용되지 않는 경우
# INDEX의 비용 (Trade-Off)
# 실행 계획(EXPLAIN)
