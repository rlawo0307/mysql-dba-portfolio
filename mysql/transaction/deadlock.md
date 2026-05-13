# Deadlock 개요
## Deadlock이란?
* 둘 이상의 트랜잭션이 서로가 보유한 lock을 기다리는 상태
* 서로가 서로를 기다리는 순환 대기(circular wait) 상태가 되어 어떤 트랜잭션도 진행할 수 없게 됨
```text
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

## MySQL의 Deadlock Detection
* InnoDB는 `Wait-for Graph`을 통해 deadlock을 감지
* lock contention이 심한 환경에서는 detection 자체가 CPU 부담이 될 수 있음
* 특히, 하나의 row에 많은 session이 동시에 접근하는 경우, detection 비용이 증가할 수 있음
* `innodb_deadlock_detect` 옵션을 통해 on/off 가능
    * 비활성 시, deadlock 즉시 감지 불가하며 lock wait timeout까지 대기
    * CPU 오버헤드를 줄이기 위한 튜닝 포인트로 사용 가능
```sql
show variables like 'innodb_deadlock_detect';
set global innodb_deadlock_detect = ON;
set global innodb_deadlock_detect = OFF;
```
### Wait-for Graph
* 트랜잭션 간의 lock 대기 관계를 그래프로 표현하는 개념적 모델
    * 별도의 graph object를 생성해서 저장하는 것이 아님
    * lock metadata를 따라 트랜잭션 간 wait chain을 탐색하여 deadlock 여부를 판단
* node : 트랜잭션
* edge : 한 트랜잭션이 다른 트랜잭션이 보유한 lock을 기다리는 관계
* 이 그래프에서 cycle이 발견되면 deadlock
```text
ex) A → B → C → A (cycle 발생 = deadlock)

TransactionA → TransactionB
TransactionB → TransactionC
TransactionC → TransactionA
```
### 감지 과정
```text
1. 트랜잭션A가 lock을 요청
2. 트랜잭션B가 lock을 보유 중이면 트랜잭션A는 대기 상태로 전환
3. InnoDB가 현재 lock wait 관계를 기반으로 wait-for graph 탐색
4. cycle 존재 여부 확인
5. deadlock이면 victim transaction을 선택하여 rollback
    * 일반적으로 변경 row 수, undo log 양, rollback cost 등을 고려하여 상대적으로 작은 트랜잭션을 선택
```
## Deadlock 확인 방법
### 최근 감지된 deadlock 확인
```sql
show engine innodb status\G
```
* [`LATEST DETECTED DEADLOCK`](../administration/show-engine-innodb-status-analysis.md) 섹션을 통해 최근 감지된 deadlock 확인 가능
    * 어떤 트랜잭션이 deadlock에 관여했는지
    * 누가 어떤 lock을 보유하는지
    * 어떤 lock을 기다리는지
    * victim transaction이 누구인지 등
### 현재 lock wait 관계 확인 [→ 모니터링 루틴](../administration/monitoring.md)
```sql
select * from information_schema.innodb_trx; -- 현재 실행 중인 트랜잭션 상태
select * from performance_schema.data_lock_waits; -- 누가 누구 때문에 기다리는지
select * from performance_schema.data_locks; -- 누가 어떤 lock을 보유/요청 중인지
```
## Deadlock 대응 및 예방 전략(★)
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
<br><br>

# Deadlock vs Lock Wait Timeout
### Deadlock
* 트랜잭션 간 circular wait 발생
* lock wait 발생 시, InnoDB가 즉시 deadlock 여부를 검사
* victim transaction을 선택해서 강제 rollback
* 1213 error 출력
```sql
ERROR 1213 (40001): Deadlock found when trying to get lock
```
### Lock Wait Timeout
* deadlock 없이 단순히 lock이 오래 유지되는 상황
* circular wait 없어도 발생 가능
* `innodb_lock_wait_timeout` 설정값 초과 시 실패
* 1205 error 출력
```sql
ERROR 1205 (HY000): Lock wait timeout exceeded
```