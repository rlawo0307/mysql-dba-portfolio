# 트랜잭션(Transaction)이란?
* [→ 트랜잭션에 대한 내용 자세히 보기](../transaction/transaction-basics.md)
* [→ autocommit, 트랜잭션 시작 시점 확인, DDL implicit commit 동작 확인 실습](../transaction/labs/transaction-implicit-commit.md)
<br><br>

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
<br><br>

# 트랜잭션 자동 커밋 설정하기
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