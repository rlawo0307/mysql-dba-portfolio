# Partition에서의 Index
* partition table에서 인덱스는 partition 단위로 나뉘어 관리됨
* 각 partition마다 독립적인 인덱스 구조
    * 논리적으로, 하나의 인덱스처럼 보임
    * 물리적으로, 동일한 인덱스 정의가 partition 개수만큼 생성됨
## Partitioned Index 종류
### Local Index
* MySQL은 local index만 지원
* 각 partition에 대해 별도의 인덱스를 생성하는 방식
* 인덱스 탐색도 partition 단위로 수행
* pruning이 되면 일부 인덱스만 탐색
* partition key 컬럼과 인덱스 컬럼이 달라도 partition 별로 local index 생성
    * 단, pruning이 안되는 경우 모든 partition을 탐색해야 함
### Global Index (미지원)
* MySQL은 global index 미지원
* 모든 partition을 통합하는 하나의 인덱스
* partition drop/add 시 유지 비용이 큼
* 전체 구조 재구성 필요
* MySQL은 ddl을 가볍게 유지하는 방향을 선택
## 인덱스 동작 흐름
* **pruning이 인덱스 탐색보다 먼저 수행됨(★)**
    * partitioned table에서의 성능은 인덱스보다 pruning이 먼저 결정함
```text
1. partition pruning
2. pruning 이후 동작
    * pruning 성공 시
        1. 대상 partition 선택
        2. 각 partition의 인덱스 탐색
        3. 결과 merge
    * pruning 실패 시
        1. 모든 partition의 인덱스 탐색
```
## 제약 조건
### pk 제약
* **pk는 반드시 partition key를 포함해야 함**
    * partition 별로 나뉜 상태에서도 pk 유일성을 보장해야하기 때문에
### fk 제약
* partition table에서 fk를 매우 제한적으로만 허용함
    * MySQL은 global index를 지원하지 않음
    * fk 검증 시 참조 row의 위치를 빠르게 찾을 수 없음
    * partition key를 모르면 어느 partition에 있는지 알 수 없음
    * 모든 partition을 탐색해야 하는 비효율 발생
* **참조 row의 partition을 특정할 수 있는 경우에만 제한적으로 허용함**
```text
* 일반적으로 아래 조건을 만족해야 함
    1. parent, child 테이블 모두 partitioned table
    2. 두 테이블의 partition key가 동일
    3. fk 컬럼이 partition key를 포함

* 위 조건을 만족하면
    * partition pruning 가능
    * 특정 partition만 탐색하여 fk 검증 가능
```