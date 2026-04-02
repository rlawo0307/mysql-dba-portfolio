# 트랜잭션 개요
## 트랜잭션(Transaction, TX) 이란?
* 하나의 논리적 작업 단위를 구성하는 SQL 연산들의 집합
* 트랜잭션에 포함된 SQL 문들은 모두 성공하거나 모두 실패해야 함
* 트랜잭션의 필요성
    * 여러 SQL을 하나의 논리적 작업 단위로 처리하기 위해
    * 작업 중 오류가 발생했을 때 전체 작업을 되돌리기 위해
    * 여러 사용자가 동시에 데이터를 수정하는 환경에서 데이터 일관성을 유지하기 위해
## 트랜잭션 속성(ACID)
* 트랜잭션이 안정적으로 동작하기 위해 보장해야 하는 4가지 속성
    * `Atomicity`(원자성)
        * 트랜잭션은 더 이상 쪼갤 수 없는 하나의 논리적 작업 단위이며
        * 그 결과는 전부 반영되거나 전부 취소되어야 함
        * 트랜잭션 내 모든 SQL문이 정상 수행되고 난 후 commit될 경우에만 변경 사항이 반영됨
        * 하나라도 실패하거나 rollback되면 모든 변경이 취소됨
        * undo log로 구현 [→ 트랜잭션 메커니즘에 대한 내용 자세히 보기](transaction-mechanism.md)
    * `Consistency`(일관성)
        * 트랜잭션 전후에 데이터베이스는 항상 정의된 규칙을 만족하는 상태로 유지되어야 함
        * 트랜잭션은 일관된 상태에서 시작하여
        * commit 이후에도 일관된 상태로 종료되어야 함
        * 기준
            * 제약 조건(pk, fk, unique, not null, check 등)
            * 트리거
            * 애플리케이션 비즈니스 로직 등
    * `Isolation`(격리성)
        * 여러 트랜잭션이 동시에 실행되더라도 각 트랜잭션이 독립적으로 실행되는 것처럼 보여야 함
        * 격리되지 않은 경우 트랜잭션의 중간 상태가 노출되어 이상현상이 발생할 수 있음
        * [isoloation level](isolation-levels.md)로 제어
        * MVCC, lock을 통해 구현
    * `Durability`(영속성)
        * commit된 데이터는 시스템 장애가 발생해도 유지되어야 함
        * redo log, checkpoint로 구현 [→ 트랜잭션 메커니즘에 대한 내용 자세히 보기](transaction-mechanism.md)
<br><br>

<center>

|ACID           |설명                            |InnoDB 구현        |
|---------------|:-----------------------------:|:-----------------:|
|**Atomicity**  |트랜잭션은 모두 성공하거나 모두 실패|undo log|
|**Consistency**|트랜잭션 전후 데이터 무결성 유지    |제약 조건, 트리거, 애플리케이션 로직|
|**Isolation** |트랜잭션 간 간섭 방지|MVCC, lock  |
|**Durability** |commit된 데이터는 장애 후에도 유지 |redo log, checkpoint|

</center>

## 트랜잭션의 흐름 [→ TCL에 대한 내용 자세히 보기](../sql-commands/sql-tcl.md)
* `begin` / `start transaction` : 트랜잭션 시작
* `commit` : 현재 트랜잭션의 변경사항을 영구적으로 저장
* `rollback` : 트랜잭션동안 발생한 변경사항을 취소
## 트랜잭션 생명주기
* `Active`
    * 트랜잭션이 시작됨
    * SQL문들이 하나씩 실행되는 중
    * 아직 commit도 rollback도 되지 않은 상태
    * 변경 내용이 아직 확정되지 않음
    * isolation level에 의해 다른 트랜잭션에서 조회 가능한 데이터 범위가 달라짐
* `Partially Committed`
    * 모든 SQL은 끝났지만 commit이 완전히 끝나지 않은 상태
    * commit 요청은 들어왔지만 redo log가 디스크에 flush되기 전
* `Committed`
    * commit이 완료되어 트랜잭션이 완전히 성공한 상태
    * redo log가 디스크에 기록되어 변경 내용이 영구적으로 보장됨
    * 이후 장애 발생 시 데이터 복구 가능
    * rollback 불가
* `Failed`
    * 트랜잭션이 더 이상 정상 진행될 수 없는 상태
    * 트랜잭션 중단
    * 아직 rollback은 수행되지 않음
* `Aborted`
    * 실패한 트랜잭션을 rollback한 상태
    * 트랜잭션이 수행한 모든 변경 사항을 취소
    * 데이터베이스를 트랜잭션 시작 전 상태로 복구
    * 복구 후 트랜잭션 비정상 종료
```text
- 정상 흐름 : Active → Partially Committed → Committed
- 실패 흐름 : Active / Partially Committed → Failed → Aborted
```