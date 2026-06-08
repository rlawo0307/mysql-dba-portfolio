# Metadata Lock(MDL)
* MySQL Server가 metadata의 일관성을 보호하기 위해 사용하는 lock
* 객체 사용 중 metadata가 변경되는 상황을 제어
* DDL과 DML 사이의 metadata 충돌 방지
* MySQL 5.5 부터 도입
* InnoDB lock(S lock, X lock, record lock, gap lock 등)과 별개의 개념
* 트랜잭션 내에서 획득한 MDL은 commit 또는 rollback 전까지 유지됨
## MDL 종류
* `Shared MDL`
    * 객체를 사용하는 작업이 획득하는 metadata lock
    * 다른 세션의 DML 수행 허용
    * DDL은 제한
    * ex) select, insert, update, delete 등의 DML
* `Exclusive MDL`
    * 객체 구조를 변경하는 작업이 획득하는 metadata lock
    * 다른 세션의 접근을 제한
    * ex) alter, drop, rename, truncate 등의 DDL
```text
Shared + Shared         → 가능
Shared + Exclusive      → 대기
Exclusive + Exclusive   → 대기
```
## MDL 조회
### performance_schema.metadata_locks
* 현재 MySQL Server에서 발생한 MDL 정보를 조회할 수 있는 시스템 테이블
* MDL 대기 상황에서 원인 세션을 식별하기 위한 핵심 지표
* DDL과 DML 사이의 metadata 충돌 여부를 확인할 수 있음
* DDL이 대기하는 원인을 분석하기 위한 주요 도구
* 주요 컬럼 
    * `object_type` : lock 대상 객체 종류 (table, schema 등)
    * `object_schema` : 객체가 속한 database
    * `object_name` : lock 대상 객체 이름
    * `lock_type` : MDL 종류 (shared_read, shared_write, exclusive 등) 
    * `lock_duration` : lock 유지 기간
    * `lock_status`
        * `granted` : lock 획득 완료
        * `pending` : lock 대기 중
## MDL 대기 시 해결 방안
### MDL 대기 상황
```sql
-- session 1
start transaction;
select * from t1; -- MDL Shared 보유
```
```sql
-- session 2
alter table t1 add column c3 int; -- MDL Exclusive 대기
                                  -- session1의 트랜잭션이 종료(commit 또는 rollback)되기 전까지 대기
```
### 해결 방안
```text 
1. processlist를 통해 대기 세션 확인
2. performance_schema.metadata_locks를 통해 MDL 상태 확인
3. MDL을 장시간 보유 중인 세션 식별
4. 해당 세션을 commit 또는 kill 
```