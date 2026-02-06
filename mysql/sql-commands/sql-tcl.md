# 트랜잭션(Transaction)이란?
* DB의 상태를 변화시키는 하나의 논리적 작업 단위
* 논리적인 이유로 여러 SQL문들을 하나의 작업으로 묶어 나누어질 수 없도록 함
    * 단, dml만 트랜잭션으로 묶을 수 있음
    * ddl, dcl은 실행 시 암묵적으로 commit을 발생시켜 현재 트랜잭션을 종료함
    * tcl은 트랜잭션을 제어하는 명령어로 트랜잭션 대상이 아님
* 규칙 (`ACID`)
    * 원자성 (Atomicity)
    * 일관성 (Consistency)
    * 격리성 (Isolation)
    * 지속성 (Durability)
* 트랜잭션의 생명주기
    * 상태
        * `Active`
            * 트랜잭션이 시작됨
            * SQL문들이 하나씩 실행되는 중
            * 아직 commit도 rollback도 되지 않은 상태
            * 변경 내용이 아직 확정되지 않음
            * Isolation Level에 의해 다른 트랜잭션에서 조회 가능한 데이터 범위가 달라짐
        * `Partially Committed`
            * 모든 SQL은 끝났지만 commit이 완전히 끝나지 않은 상태
            * commit 요청은 들어왔지만 실제로 디스크에 영구 반영되기 전
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
    * 정상 흐름 : `Active → Partially Committed → Committed`
    * 실패 흐름 : `Active / Partially Committed → Failed → Aborted`
## 원자성 (Atomicity)
* 트랜잭션은 더 이상 쪼갤 수 없는 하나의 논리적 작업 단위이며
* 그 결과는 전부 반영 또는 전부 취소 둘 중 하나뿐임
    * 트랜잭션 내 모든 SQL문이 정상 수행 후 commit될 경우에만 변경 사항이 반영됨
    * 하나라도 실패하거나 rollback되면 모든 변경이 취소됨
* `undo log`로 구현됨
    1. 변경 전 데이터를 undo log에 기록
    2. 실제 데이터 페이지 변경
    3. 트랜잭션 종료 후
        * rollback 시 : undo log를 적용해 변경 내용을 원래 상태로 복구
        * commit 시 : 변경 내용이 확정됨. undo log는 즉시 삭제되지 않음
    4. 이후 동작
        * Consistent Read(MVCC)
            * 다른 트랜잭션이 Read View에 따라 undo log를 참조해 과거 버전의 데이터를 조회할 수 있음
        * undo log 정리
            * 참조 중인 Read View가 없을 경우 purge 과정에서 정리됨
## 일관성 (Consistency)
* 트랜잭션 전후에 데이터베이스는 항상 정의된 규칙을 만족하는 상태로 유지되어야 함
    * 트랜잭션은 일관된 상태에서 시작하여
    * commit 이후에도 일관된 상태로 종료되어야 함
* 기준
    * 제약조건 (pk, fk, unique, not null, check 등)
    * 트리거
    * 애플리케이션 비즈니스 로직 등
## 격리성 (Isolation)
* 여러 트랜잭션이 동시에 실행되더라도 각 트랜잭션이 독립적으로 실행한 것처럼 보이도록 함
* 격리되지 않은 경우 트랜잭션의 중간 상태가 노출되어 `이상현상`이 발생할 수 있음
    * `Dirty Read` : commit되지 않은 데이터를 다른 트랜잭션이 읽음
    * `Non-Repeatable Read` : 같은 데이터를 두 번 읽었는데 값이 다름
    * `Phantom Read` : 같은 조건으로 조회했는데 행의 개수가 달라짐
* Isolation Level
    * 동시에 실행되는 트랜잭션들이 **서로의 변경을 어디까지 볼 수 있게 할 것인가**를 정하는 기준
    * 격리 수준 ↑ → 안정성 ↑, 동시성 ↓, 성능 ↓
    * 격리 수준 ↓ → 성능 ↑, 이상현상 가능성 ↑
    * 종류
        * `READ UNCOMMITTED`
            * commit, rollback 여부와 상관없이 다른 트랜잭션에서 조회 가능
        * `READ COMMITTED`
            * commit이 완료된 데이터만 다른 트랜잭션에서 조회 가능
            * select 쿼리마다 새로운 Read View 생성
        * `REPEATABLE READ`
            * 같은 트랜잭션 안에서는 항상 같은 결과를 보장
            * 트랜잭션 시작 후 처음 생성된 Read View를 끝까지 유지
            * 표준 SQL 정의상, Phantom Read 발생 가능
                * InnoDB의 경우, Next-Key Lock으로 Phantom Read까지 방지
        * `SERIALIZABLE`
            * 모든 읽기 작업에 read lock을 획득해야함
            * 가장 단순하면서 가장 엄격한 격리 수준
            * 동시 처리 성능이 매우 떨어짐
<center>

|                    |Dirty Read|Non-Repeatable Read|Phantom Read|
|--------------------|:--------:|:-----------------:|:----------:|
|**READ UNCOMMITTED**|O|O|O|
|**READ COMMITTED**  |X|O|O|
|**REPEATABLE READ** |X|X|O (InnoDB의 경우 X)|
|**SERIALIZABLE**    |X|X|X|
</center>

* `MVCC`, `undo log`, `lock`으로 구현됨
## 지속성 (Durability)
* commit된 데이터는 장애가 발생하더라도 반드시 유지되어야 함
* `redo log`, `checkpoint`로 구현
<br>

# 트랜잭션 시작하기
* 명시적으로 트랜잭션을 시작 (state=`Active`)
* 이후 실행되는 dml들은 하나의 트랜잭션으로 묶임
* `autocommit=0`과 동일한 효과
* isolation level은 트랜잭션 시작 시점에 결정됨
```sql
start transaction;
```
```sql
begin;
``` 
<br>

# 트랜잭션 종료하기
### COMMIT
* 트랜잭션에서 수행한 모든 변경 사항을 영구 반영 (state=`Committed`)
* redo log가 디스크에 기록되어 변경 내용이 영구적으로 보장됨
* 이후 rollback 불가
```sql
start transaction;
...
commit;
```
### ROLLBACK
* 트랜잭션 내 모든 변경 사항을 취소 (state = `Failed → Aborted`)
    * 마지막 commit 이후, 아직 commit 되지 않은 변경사항만 복구 가능
* undo log를 이용하여 변경 전 상태로 복구
```sql
start transaction;
쿼리1;
쿼리2;
rollback; -- 쿼리1, 쿼리2 모두 취소됨
```
```sql
start transaction;
쿼리1;
쿼리2;
commit;
rollback;  -- 이미 커밋됐으므로 쿼리1, 쿼리2는 취소되지 않음
```
# 트랜잭션 중간 제어
### SAVEPOINT
* 트랜잭션 내부에 중간 복구 지점 생성 (state=`Active`)
* 긴 트랜잭션에서 부분 취소가 필요할 때 사용
* 여러개의 savepoint 생성 가능
* 트랜잭션이 종료되면(commit 또는 rollback) 모든 savepoint는 자동으로 제거됨
```sql
start transaction;
쿼리1;
savepoint sp1;
쿼리2;
savepoint sp2;
쿼리3;
commit;
```
### ROLLBACK TO SAVEPOINT
* 특정 savepoint 이후의 변경만 취소
* 트랜잭션 자체는 종료되지 않음 (state=`Active`)
* 지정한 savepoint 이전의 변경 내용은 유지됨
* undo log 중 해당 시점 이후의 변경만 롤백
```sql
start transaction;
쿼리1;
savepoint sp1;
쿼리2;
savepoint sp2;
쿼리3;
rollback to savepoint sp1; -- 쿼리2, 쿼리3은 취소됨. 쿼리1은 유지
commit; -- 쿼리1만 커밋됨
```
### RELEASE SAVEPOINT
* 생성한 savepoint 제거
* 이후 해당 savepoint로 rollback 불가
```sql
start transaction;
쿼리1;
savepoint sp1;
쿼리2;
savepoint sp2;
쿼리3;
release savepoint sp1; -- sp1 삭제
rollback to savepoint sp1; -- sp1로 롤백 불가
rollback to savepoint sp2; -- 쿼리3은 취소됨. 쿼리1, 쿼리2는 유지
commit; -- 쿼리1, 쿼리2만 커밋됨
```
### 존재하는 savepoint 조회하기
* MySQL에서는 현재 트랜잭션 내에서 존재하는 savepoint 목록을 직접 조회하는 방법은 지원하지 않음
* 간접적으로 관리하는 방법
    * 애플리케이션 코드에서 savepoint 이름을 변수나 로그로 관리
    * 저장해둔 savepoint를 기반으로 `rollback to savepoint` 실행하기
# 트랜잭션 자동 커밋 설정하기 (SET AUTOCOMMIT)
* 기본적으로 MySQL은 autocommit 모드
    * 각 SQL문이 실행될 때마다 자동으로 즉시 커밋됨
    * 한 개의 SQL문이 하나의 트랜잭션처럼 동작
    * rollback 불가능
* autocommit이 꺼져있으면 여러 SQL문을 하나의 트랜잭션으로 직접 제어 가능
* 중요한 연산 시 데이터 무결성 유지를 위해 트랜잭션을 직접 제어하는 것이 좋음
```sql
set autocommit = 0; -- off (트랜잭션 직접 제어)
set autocommit = 1; -- on (자동 커밋)
```