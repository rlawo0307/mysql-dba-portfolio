```
[참고]

monitoring          : 현재 상태 관찰 및 문제 징후 감지
audit               : 사용자 행위 및 접근 기록 추적
logging             : 시스템 이벤트 및 과거 기록 저장
backup / dump       : 데이터 보존 (장애 대비)
recovery            : 데이터 복구 (과거 상태로 복원)
schema versioning   : 스키마 변경 이력 관리
```
<br>

# Logging이란?
* MySQL에서 발생하는 다양한 이벤트를 기록하는 기능
* 문제 분석, 장애 추적, recovery, 사용자 행위 분석의 기반이 되는 데이터
* 사용하는 경우
    * DB가 비정상 종료되었을 때 원인 파악 → `error log`
    * 느린 쿼리나 비효율적인 쿼리 추적 → `slow query log`
    * 애플리케이션이 실제로 보낸 SQL 추적 → `general query log`
    * 특정 시점까지 데이터 복구(PITR) → `binary log`
    * replication 장애 분석 및 데이터 변경 흐름 추적 → `binary log` / `relay log`
    * crash 이후 committed transaction 복구 → `redo log`
    * rollback / MVCC old version 관리 → `undo log`
* 특징
    * 과거 이벤트를 기반으로 분석 가능
    * 장애 원인 추적 가능
    * 일부 로그를 통해 사용자 행위 추적 가능
* 한계
    * 실시간 상태는 알 수 없음
    * 로그 종류와 기록량에 따라 성능에 영향을 줄 수 있음
<br><br>

# Log 저장 방식
* MySQL의 대부분의 log는 저장 방식이 정해져 있음
* 일부 로그(`general log`, `slow query log`)는 저장 위치를 선택할 수 있음
### TABLE 방식
* 로그를 MySQL 내부 테이블에 저장
* 장점
    * SQL로 바로 조회 가능
    * 필터링 및 검색이 편함
    * 애플리케이션 없이 바로 분석 가능
* 단점
    * DB 내부 write 발생
    * 로그가 많아지면 테이블이 커짐
    * 성능에 영향을 줄 수 있음
```sql
set global log_output='TABLE'; -- table 방식으로 설정
show variables like 'log_output'; -- 로그 저장 방식 확인
select * from mysql.general_log; -- SQL로 로그 조회
```
### FILE 방식
* 로그를 OS 파일에 저장
* 장점
    * OS에서 바로 확인 가능
    * 실시간 tail 가능
    * 외부 로그 수집 시 시스템 연동이 쉬움
* 단점
    * SQL 기반 조회 / 검색이 어려움
    * 로그 파싱 도구가 필요할 수 있음
    * 파일 용량 관리 필요
    * 파일 권한 및 접근 관리 필요
```sql
set global log_output='FILE'; -- file 방식으로 설정
show variables like 'log_output'; -- 로그 저장 방식 확인
show variables like 'general_log_file'; -- general log 경로 확인
```
```bash
tail -f <general log file 경로> # 경로에 존재하는 general log 내용 확인
```
<br>

# Log 종류
## Error Log
* 특징
    * MySQL 서버 운영 중 발생하는 주요 시스템 이벤트와 오류를 기록하는 로그
    * 서버의 시작, 실행, 종료 과정에서 발생한 문제를 기록
    * 가장 기본적인 운영 로그
    * 장애 분석 시 1순위로 확인하는 로그
* 기록 내용
    * 서버 시작 / 종료 메시지
    * 비정상 종료 관련 정보
    * 설정 오류
    * InnoDB 관련 오류 / 경고
    * replication 관련 오류 및 상태 변화
    * 기타 경고 및 정보성 메시지
### error log 파일 경로 및 내용 확인
```sql
show variables like 'log_error'; -- error log 파일 경로 확인
```
```bash
tail -f <error log file 경로> # 위에서 조회된 경로에 위치하는 error log 내용 확인
```
## General Log
* 특징
    * 클라이언트가 서버에 보낸 요청을 기록하는 로그
    * 클라이언트 연결 / 해제와 클라이언트가 보낸 각 SQL문을 기록
    * 누가 어떤 쿼리를 보냈는지 확인할 수 있음
    * 실행 결과가 아닌 요청 자체를 기록
    * select 포함 모든 SQL 요청이 기록됨
    * 디버깅 및 쿼리 추적에 유용
    * 로그 양이 매우 빠르게 증가하고 성능 부담이 커질 수 있어 운영 환경에서는 상시 활성화를 지양
* 기록 내용
    * 클라이언트-서버 연결 / 해제 메시지
    * 클라이언트가 보낸 SQL문
    * prepare / execute 요청
### general log 설정 및 확인
```sql
set global general_log = ON; -- general log 활성화 (활성화 시점 이후부터 기록)

show variables like 'general_log'; -- 활성화 여부 확인
show variables like 'general_log_file'; -- 경로 확인
```
```bash
tail -f <general log file 경로> # 경로에 존재하는 general log 내용 확인
```
## Slow Query Log
* 특징
    * 느린 쿼리를 기록하는 성능 분석용 로그
    * 성능 튜닝 시 가장 실용적으로 사용하는 로그 중 하나
    * 오래 걸리거나 비효율적인 쿼리를 식별하는 데 사용
    * 옵션을 통해 기록 대상을 제어할 수 있음
        * `long_query_time`
            * 설정 시간 이상 실행된 쿼리 기록
            * 너무 낮으면 로그 양이 급격히 증가할 수 있음
            * 너무 높으면 slow query 추적 불가
        * `min_examined_row_limit`
            * 설정한 row 수 이상 검사한 쿼리만 기록
            * 비효율적인 full scan 쿼리 필터링에 유용
        * `log_queries_not_using_indexes`
            * 인덱스 미사용 쿼리 기록
            * 실행 시간, row 검사 조건과 별개로 인덱스를 사용하지 않으면 무조건 기록
            * 느린 쿼리보다는 나쁜 쿼리를 기록하는 로그 옵션
            * 작은 테이블이나 인덱스가 필요 없는 쿼리도 기록되기 때문에 로그 양이 증가할 수 있음
* 기록 내용
    * 느리게 실행된 SQL문
    * 쿼리 실행 시간
    * lock 대기 시간
    * 검사한 row 수
    * 반환한 row 수
    * 옵션에 따라 인덱스 미사용 쿼리 등
### slow query log 설정 및 확인
```sql
set global slow_query_log = ON;

show variables like 'slow_query_log';
show variables like 'slow_query_log_file'; -- 경로 확인
```
```bash
tail -f <slow query log file 경로> # 경로에 존재하는 slow query log 내용 확인
```
## [Binary Log](../availability/binlog-and-GTID.md)
* 특징
    * 데이터 변경 이벤트를 기록하는 로그
    * 데이터 변경 내용을 기록하는 logical log
    * replication과 point-in-time recovery(PITR)의 핵심 로그
    * source 서버에서 생성됨
    * 일반 select 조회는 기록되지 않음
    * crash recovery용 로그는 아님
* 기록 내용
    * insert / update / delete
    * DDL
    * 트랜잭션 commit 정보
    * GTID 정보 (GTID mode 사용 시)
### binary log 설정 및 확인
* binary log는 text 로그가 아닌 MySQL 전용 binary format으로 저장됨
* MySQL에서 제공하는 parser(`mysqlbinlog`)로 해석해야 사람이 읽을 수 있음
```sql
show variables like 'log_bin'; -- binary log 활성화 여부 확인
show variables like 'binlog_format'; -- 어떤 방식으로 binlog를 기록하는지 확인(statement/row/mixed)

show master status; -- 현재 source 서버의 활성 binary log 상태 확인
show binary logs; -- 서버에 존재하는 binary log 목록 전체 확인
```
```bash
mysqlbinlog <binary log file 경로>
```
## Relay Log
* 특징
    * replica가 source의 binary log를 받아 저장하는 로그
    * replication 과정에서 중간 저장소 역할을 하는 로그
    * replica SQL thread가 relay log를 읽어 변경 사항을 적용
    * replication 장애 분석 시 중요한 로그
    * replication / PITR 용 로그는 아님
* 기록 내용
    * source에서 전달받은 binary log event
    * DDL / DML 이벤트
    * 트랜잭션 commit 정보
    * GTID 정보 (GTID mode 사용 시)
### relay log 확인
```sql
show replica status\G -- replication 전체 상태 확인
show relaylog events; -- relay log 내부 event 목록 확인
```
```bash 
mysqlbinlog <relay log file 경로> # MySQL parser로 해석
```
## [Redo Log](../transaction/transaction-mechanism.md)
* 특징
    * InnoDB의 [crash recovery](../availability/crash-recovery.md)를 위한 내부 로그
    * data page 변경 내용을 기록하는 physical log
    * WAL(Write-Ahead Logging) 기반으로 동작
    * committed transaction 복구에 사용
    * replication / PITR 용 로그는 아님
### redo log 확인
* redo log는 InnoDB 내부 메커니즘으로, 파일을 열어서 확인할 수 없음
* `show engine innodb status`를 통해 간접적으로 확인만 가능
```sql
show engine innodb status; -- InnoDB 내부 상태 종합 진단 리포트
```
## [Undo Log](../transaction/transaction-mechanism.md)
* 특징
    * rollback과 MVCC를 위한 InnoDB 내부 로그
    * 변경 이전 버전 row를 저장하는 내부 로그
    * uncommitted transaction rollback에 사용
    * consistent read(MVCC)에서 과거 버전 row를 제공
### undo log 확인
* undo log는 InnoDB 내부 recovery 메커니즘으로, 파일을 열어서 확인할 수 없음
* `show engine innodb status`를 통해 간접적으로 확인만 가능
```sql
show engine innodb status;
```