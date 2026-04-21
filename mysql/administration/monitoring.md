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
* **모니터링은 개별 요소가 아니라, 서로 연관된 상태를 종합적으로 해석해야 함**
* 특징
    * 현재 시점의 DB 상태를 즉시 확인할 수 있음
    * DB 내부 동작을 직접 관찰할 수 있음
    * 문제 발생 시 즉시 원인 분석이 가능함
* 한계
    * 현재 상태만을 보여줌 (이미 종료된 쿼리는 확인할 수 없음)
    * 순간적인 상태를 캡처하는 방식으로, 지속적인 추적에는 한계가 있음 → 반복적인 관찰 필요
    * 과거 분석을 위해서는 logging과 함께 사용해야 함
* 모니터링 대상
    * connection 상태
    * 실행 중인 쿼리
    * 트랜잭션 상태
    * 락 경합 및 대기 상태
    * 자원 사용 상태(CPU, 메모리 등)
    * InnoDB 엔진 내부 상태
<br><br>

# 모니터링 도구 [→ 모니터링 루틴 및 도구 사용 실습](labs/monitoring-routine.md)
```text 
[모니터링 루틴]

1. processlist                          → 이상 감지
2. innodb_trx                           → 트랜잭션 확인
3. data_lock_waits                      → blocking 관계 확인
4. data_locks                           → lock 대상 및 종류 확인
5. show engine innodb status            → lock 구조 및 상세 분석
6. sys schema                           → 전체 결과를 빠르게 확인
```
## processlist
* 현재 MySQL 서버에 연결된 모든 세션(=thread)과 그 상태를 확인할 수 있음
    * 누가 접속해 있고
    * 어떤 쿼리를 실행 중이고
    * 어떤 상태로 동작 중인지
* 현재 상태를 빠르게 확인하는 1차 모니터링 도구
* 문제 발생 시 가장 먼저 확인해야 하는 지표
```text
Id | User | Host | db | Command | Time | State | Info
```
* `id` : 세션(=thread) id (kill 시 사용)
* `User` : 접속한 사용자
    * `system user` 는 MySQL 내부 thread
* `Host` : 접속한 클라이언트 위치
* `db` : 현재 사용 중인 데이터베이스
* `Command` : 현재 thread 상태 (큰 동작 분류)
    * `Query` : 쿼리 실행 중
    * `Sleep` : 아무 것도 안하고 대기
    * `Connect` : 연결 관련 작업 중
    * `Daemon` : 내부 background thread
* `Time` : 현재 상태(State)를 유지한 시간(초)
* `State` : 현재 thread가 수행 중인 구체적인 작업
    * `Locked` : lock 대기 상태
    * `Sending data` : 데이터 읽기/처리 중
    * `Waiting for ...` : 특정 이벤트 대기
    * `Replica has read all relay log` : replication idle 상태
* `Info` : 실제 실행 중인 SQL
## information_schema.innodb_trx
* 현재 InnoDB 엔진에서 실행 중인 모든 트랜잭션 상태를 조회할 수 있는 시스템 테이블
    * 어떤 트랜잭션이 실행 중인지
    * 어떤 트랜잭션이 lock을 기다리고 있는지
    * 어떤 쿼리가 문제를 발생시키고 있는지 등
* 트랜잭션 단위로 상태를 확인하는 모니터링 테이블
* lock wait, blocking 관계 분석 시 핵심적으로 사용하는 지표
* 문제 원인을 분석하기 위한 2차 분석 도구
* 주요 확인 포인트
    * `trx_state` : 현재 트랜잭션 상태
        * `RUNNING` : 정상 실행 중
        * `LOCK WAIT` : lock 때문에 대기 중
    * `trx_wait_started` : lock wait이 시작된 시점
    * `trx_mysql_thread_id` : 이 트랜잭션을 실행 중인 세션(thread) id
    * `trx_query` : 현재 실행 중인 SQL (단, 실제 lock을 획득한 쿼리가 아닐 수 있음)
    * `trx_rows_modified` : 이 트랜잭션이 변경한 row 수
        * `0` : 아직 작업을 못함. 완전히 막힘
        * `>0` : 일부 작업 수행 후 진행 중
## performance_schema.data_lock_waits
* 현재 InnoDB에서 발생한 lock wait 관계를 조회할 수 있는 시스템 테이블
    * 어떤 트랜잭션이 lock을 기다리고 있는지
    * 어떤 트랜잭션이 해당 lock을 보유하고 있는지
    * 어떤 트랜잭션 간에 blocking이 발생하고 있는지 등
* lock wait 상황에서 원인 트랜잭션을 식별하기 위한 핵심 지표
* waiting 트랜잭션과 blocking 트랜잭션의 관계를 직접 연결해주는 지표
* 문제 원인을 확정하기 위한 3차 분석 도구
* 주요 확인 포인트
    * `REQUESTING_ENGINE_TRANSACTION_ID` : lock을 기다리는 트랜잭션 id
    * `BLOCKING_ENGINE_TRANSACTION_ID` : lock을 잡고 있는 트랜잭션 id
## performance_schema.data_locks
* 현재 InnoDB에서 설정된 모든 lock 정보를 조회할 수 있는 시스템 테이블
    * 어떤 테이블/인덱스에 lock이 걸려 있는지
    * 어떤 row에 lock이 걸려 있는지
    * lock의 종류
* lock이 걸린 구체적인 위치(테이블, 인덱스, 값)를 확인하기 위한 핵심 지표
* 문제 원인을 확정하기 위한 3차 분석 도구
* 주요 확인 포인트
    * `ENGINE_TRANSACTION_ID` : lock을 잡고 있는 트랜잭션 id
    * `OBJECT_SCHEMA`, `OBJECT_NAME` : lock이 걸린 테이블
    * `INDEX_NAME` : lock이 걸린 인덱스
    * `LOCK_TYPE` : lock이 걸리는 대상(table/record) 
    * `LOCK_MODE` : lock 종류
    * `LOCK_STATUS` : lock의 현재 상태
    * `LOCK_DATA` : lock이 걸린 실제 row 값
## show engine innodb status
* InnoDB 엔진의 내부 상태를 종합적으로 확인할 수 있는 명령어
    * 현재 실행 중인 트랜잭션 상태
    * lock wait 및 blocking 상황
    * 정확히 어떤 row에 어떤 lock이 걸려 있는지
    * deadlock 발생 내역 등
* lock, 트랜잭션, deadlock 등 상세 분석에 사용되는 핵심 지표
* 문제 상황의 내부 구조를 확인하기 위한 최종 분석 도구
* 다른 도구로 확인한 내용을 기반으로, 실제 lock 구조를 최종적으로 검증할 때 사용
## sys schema
* performance_schema를 기반으로 한 고수준 뷰 모음
* 내부 정보를 사람이 보기 쉽게 가공하여 제공
* 복잡한 join 없이 lock, 트랜잭션 상태를 빠르게 확인 가능
* 운영 환경에서 빠른 문제 파악을 위해 사용되는 보조 도구
* 내부 구조 분석용이 아닌, 결과를 빠르게 확인할 때 사용