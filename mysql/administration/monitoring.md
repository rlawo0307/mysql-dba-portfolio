```
[참고]

monitoring / audit  : 현재 상태 및 사용자 행위 추적
logging             : 과거 이벤트 기록
backup / dump       : 데이터 보존 (장애 대비)
recovery            : 데이터 복구 (과거 상태로 복원)
schema versioning   : 스키마 변경 이력 관리
```
<br>

# Monitoring이란?
* MySQL 내부에서 현재 어떤 일이 일어나고 있는지 관찰하는 행위
* DB 내부 상태를 실시간으로 파악하고 문제 원인을 추적하기 위해 사용
* 특징
    * 현재 시점의 DB 상태를 즉시 확인할 수 있음
    * DB 내부 동작을 직접 관찰할 수 있음
    * 문제 발생 시 즉시 원인 분석이 가능함
* 한계
    * 현재 상태만을 보여줌 (이미 종료된 쿼리는 확인할 수 없음)
    * 순간적인 상태를 캡처하는 방식으로, 지속적인 추적에는 한계가 있음 → 반복적인 관찰 필요
    * 과거 분석을 위해서는 logging과 함께 사용해야 함
## 모니터링 대상
* 실행 중인 쿼리
* 트랜잭션 상태
* 락 경합 및 대기 상태
* 자원 사용 상태(CPU, 메모리 등)
* InnoDB 엔진 내부 상태
#### ※ 모니터링은 개별 요소가 아니라, 서로 연관된 상태를 종합적으로 해석해야 함
## 모니터링 도구
* `processlist` : 실행 중인 쿼리 확인
* `information_schema` : 트랜잭션 및 메타 정보 확인
* `performance_schema` : 내부 동작 및 대기 정보 확인
* `sys schema` : 모니터링 용 view 제공
* `innodb_trx` : InnoDB 트랜잭션 상태 확인
* `show engine innodb status` : InnoDB 내부 상태 및 deadlock 정보 확인