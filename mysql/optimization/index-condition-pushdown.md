# Index Condition Pushdown(ICP)
* index scan 과정에서 일부 where 조건을 스토리지 엔진 단계에서 미리 적용하는 최적화 기법
### 특징
* 인덱스 단계에서 where 조건 일부를 먼저 필터링
* data page 접근 전 where 조건 필터링
* 불필요한 data page 접근 감소
* explain 실행 시, Extra에 `Using index condition` 출력
### 발생 조건
* secondary index를 사용할 때
* where 조건에 인덱스 컬럼이 포함되어 있을 때
* 인덱스에 있는 컬럼만으로 평가 가능한 조건이 존재할 때
### 동작 원리
```text
1. secondary index 탐색
2. 인덱스 컬럼을 사용한 where 조건으로 row를 먼저 필터링
3. 조건을 통과한 row만 pk lookup 수행
4. data page 접근 최소화
```
### 장점
* data page 접근 감소 → 랜덤 I/O 감소
* buffer pool에 불필요한 data page 적재 감소
* 성능 향상 (특히 range scan에서 효과가 큼)
* large table에서 효과 극대화
### 한계
* 모든 조건이 pushdown 되는 것은 아님
    * 인덱스에 없는 컬럼은 pushdown 불가
    * 함수/연산이 포함된 조건은 pushdown이 제한될 수 있음
* pk 기반 접근에서는 효과가 제한적
    * clustered index는 data page 자체이므로 pushdown의 이점이 크지 않음
* covering index보다 효과가 떨어짐
    * ICP는 data page 접근을 줄이는 최적화 기법으로, 조회 컬럼에 접근하기 위해서는 결국 data page를 읽어야 함
    * data page 접근 자체를 제거하는 것은 covering index