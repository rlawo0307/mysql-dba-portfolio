# MySQL 운영 관리 (MySQL Administration)
이 디렉터리는 Linux 환경에서 수행한 MySQL 데이터베이스 운영 관리(DBA) 작업을 정리합니다.
<br><br>

# 목적
이 문서는 단순 SQL 사용이나 애플리케이션 개발 관점이 아닌, DBA 관점에서의 MySQL 운영 경험을 정리하는 것을 목표로 합니다.

각 항목은 실제 설정 변경, 명령어 실행, 결과 확인을 기반으로 구성되며, 운영 안정성과 유지보수 관점에서 설명할 예정입니다.
<br><br>

# 폴더 구성
## administration
* 데이터베이스 관리 - [database-basic-operations.md](administration/database-basic-operations.md)
* MySQL 설치 및 초기 설정 - [installation.md](administration/installation.md)
* 사용자 관리 - [user-account.md](administration/user-account.md)
### labs
* MySQL host 매칭 동작 확인 - [mysql-user-host-matching.md](administration/labs/mysql-user-host-matching.md)
## core-concepts
* MySQL 내장함수 - [builtin-functions.md](core-concepts/builtin-functions.md)
* 제약조건 - [constraint-theory.md](core-concepts/constraint-theory.md)
* 데이터 타입 - [data-types-theory.md](core-concepts/data-types-theory.md)
* MySQL SQL Compiler Engine - [sql-compiler-engine.md](core-concepts/sql-compiler-engine.md)
* InnoDB 물리적 저장 구조 - [storage-architecture.md](core-concepts/storage-architecture.md)
### labs
* Foreign Key 동작 분석 - [fk-behavior.md](core-concepts/labs/fk-behavior.md)
## index
* 인덱스를 타지 않는 쿼리 패턴 - [index-antipatterns.md](index/index-antipatterns.md)
* 인덱스 기본 - [index-basics.md](index/index-basics.md)
* 인덱스 힌트 - [index-hint.md](index/index-hint.md)
### labs
* Index Scan 유형 분석 - [01-explain-index-scan-types.md](index/labs/01-explain-index-scan-types.md)
* Full Table Scan vs Index Scan - [02-full-table-scan-vs-index-scan.md](index/labs/02-full-table-scan-vs-index-scan.md)
* 옵티마이저의 인덱스 선택 - [03-index-selection.md](index/labs/03-index-selection.md)
* 인덱스 성능 실험 - [04-index-performance-experiment.md](index/labs/04-index-performance-experiment.md)
* 인덱스 성능 실험 2 - [05-index-performance-experiment-2.md](index/labs/05-index-performance-experiment-2.md)
* 복합 인덱스 순서 실험 - [06-composite-index-ordering.md](index/labs/06-composite-index-leftmost-rule.md)
* Filesort vs Index Ordering - [07-filesort-vs-index-ordering.md](index/labs/07-filesort-vs-index-ordering.md)
## optimization
* 실행 계획 이해 - [explain.md](optimization/explain.md)
* 암시적 형 변환 - [implicit-type-conversion.md](optimization/implicit-type-conversion.md)
* join 기본 - [join-internal.md](optimization/join-internal.md)
* LIMIT 쿼리 이해 - [limit.md](optimization/limit.md)
* 옵티마이저와 통계정보 - [optimizer-statistics.md](optimization/optimizer-statistics.md)
## sql-commands
* SQL 명령어 - [sql-command-overview.md](sql-commands/sql-command-overview.md)
* DCL - [sql-dcl.md](sql-commands/sql-dcl.md)
* DDL - [sql-ddl.md](sql-commands/sql-ddl.md)
* DML - [sql-dml.md](sql-commands/sql-dml.md)
* TCL - [sql-tcl.md](sql-commands/sql-tcl.md)
## transaction
* 트랜잭션 기본 개념 - [transaction-basics.md](transaction/transaction-basics.md)
* 격리 수준 - [isolation-levels.md](transaction/isolation-levels.md)
* 트랜잭션 메커니즘 - [transaction-mechanism.md](transaction/transaction-mechanism.md)
* Locking - [locking.md](transaction/locking.md)
* Deadlock - [deadlock.md](transaction/deadlock.md)