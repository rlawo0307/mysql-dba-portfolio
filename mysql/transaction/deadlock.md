# Deadlock 개요
## Deadlock이란?
* 둘 이상의 트랜잭션이 서로가 보유한 lock을 기다림
* 서로가 서로를 기다리는 순환 대기(circular wait) 상태가 되어 어떤 트랜잭션도 진행할 수 없게됨
```sql
ex) TransactionA : row1 lock 획득
    TransactionB : row2 lock 획득
    TransactionA : row2 lock 요청 (대기) -- A는 B가 가진 lock을 기다림
    TransactionB : row1 lock 요청 (대기) -- B는 A가 가진 lock을 기다림
```
## Deadlock 발생 조건
* 아래 4가지를 모두 만족할 때 deadlock 발생
    * `Mutual Exclusion`(상호 배제)
        * 하나의 자원은 한 번에 하나의 트랜잭션만 사용할 수 있음
    * `Hold and Wait`(점유 대기)
        * 트랜잭션이 이미 lock을 보유한 상태로 다른 lock을 기다리는 상황
    * `No Preemption`(비선점)
        * lock을 강제로 빼앗을 수 없음
        * lock을 가진 트랜잭션이 commit/rollback해야만 lock 해제
    * `Circular Wait`(순환 대기)
        * 트랜잭션들이 원형으로 서로의 lock을 기다리는 구조

## MySQL의 Deadlock 감지 방식
* InnoDB는 `Wait-for Graph`을 통해 deadlock을 감지
* Wait-for Graph
    * 트랜잭션 간의 lock 대기 관계를 그래프로 표현
    * node : 트랜잭션
    * edge : 한 트랜잭션이 다른 트랜잭션이 보유한 lock을 기다리면 edge 생성
    * 이 그래프에서 cycle이 발견되면 deadlock
```sql
ex) A → B → C → A (cycle 발생 = deadlock)

TransactionA → TransactionB
TransactionB → TransactionC
TransactionC → TransactionA
```
* 감지 과정
    1. 트랜잭션A가 lock을 요청
    2. 트랜잭션B가 lock을 보유 중이면 트랜잭션A는 대기 상태로 전환
    3. InnoDB가 wait-for graph 생성
    4. cycle 존재 여부 탐지
    5. deadlock이면 rollback cost가 가장 적은 트랜잭션을 rollback
        * 일반적으로 변경한 row수가 적거나 undo log 크기가 작은 쪽 rollback
        * rollback된 트랜잭션을 `Victim Transaction`이라고 함
## Deadlock 해결 전략(★)
* deadlock을 완전히 제거하기는 어렵지만 줄일 수 있는 방법이 있음
    * 항상 동일한 순서로 데이터에 접근하기
    * 트랜잭션을 짧게 유지하기
        * 트랜잭션이 길어질수록 → lock 유지 시간 증가 → deadlock 가능성 증가
    * 필요한 row에만 lock 걸기
        * 불필요한 많은 lock → 충돌 확률 증가 → deadlock 가능성 증가
    * 적절한 인덱스 사용하기
        * lock이 걸리는 범위를 줄일 수 있음
    * 애플리케이션에서 retry 처리하기
        * 보통 애플리케이션에서 deadlock 발생 → rollback → 재시도 패턴 사용
