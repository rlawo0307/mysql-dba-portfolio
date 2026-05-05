# Optimizer Trace
* MySQL 옵티마이저가 실행 계획을 선택하는 내부 과정을 JSON 형태로 기록하는 기능
* 옵티마이저가 왜 이 실행 계획을 선택했는지를 보여주는 내부 분석 도구
* explain이 실행 계획 결과를 보여준다면, optimizer trace는 선택 과정과 이유를 보여줌
* 옵티마이저의 의사결정 과정 추적 가능
    * access path 선택 과정
    * index 선택 및 제외 이유
    * cost 계산 과정
* 세션 단위로 활성화 가능
* 사용하는 경우
    * 인덱스를 안타는 이유 분석이 필요할 때
    * 인덱스 선택 기준을 확인할 때
    * cardinality / histogram 영향 분석이 필요할 때
    * 인덱스 추가 전/후 비교
    * 힌트 사용 효과 확인
    * 그 외 성능 분석 및 튜닝이 필요할 때
* 주의할 점
    * trace 결과는 매우 크기 때문에 필요할 때만 활성화하기
    * 최신 쿼리만 저장되므로 여러 쿼리 분석 시 주의가 필요
## 사용법
### optimizer trace 활성화
```sql
set optimizer_trace = "enabled=on";
```
### 분석할 쿼리 실행
```sql
select * from t1 where c1 = 10;
```
### trace 결과 조회
```sql
select * from information_schema.optimizer_trace\G
```
## 결과 구조
```JSON
{
  "steps": [
    {
      "join_preparation": { ... }
    },
    {
      "join_optimization": {
        "rows_estimation": { ... },
        "considered_execution_plans": [ ... ]
      }
    }
  ]
}
```
* `join_preparation` : 쿼리 파싱 및 전처리 단계. 서브쿼리 변환, 뷰 확장 등을 수행
* `join_optimization` : 실행 계획을 결정하는 단계
    * `rows_estimation`
        * 각 조건의 row 수 추정 과정
        * cardinality / histogram 사용 여부 확인 가능
    * `considered_execution_plans`
        * 옵티마이저가 고려한 실행 계획 목록
        * 어떤 access path를 비교했는지
        * 각 path의 cost
        * 최종 선택된 path
    * `chosen_plan`
        * 최종 선택된 실행 계획