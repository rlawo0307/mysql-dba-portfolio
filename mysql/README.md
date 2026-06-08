# MySQL 운영 관리 (MySQL Administration)
* Linux 환경에서 수행한 MySQL 데이터베이스 운영 관리(DBA) 작업을 정리한 레포지토리입니다.
* 각 문서는 단순 SQL 사용이 아닌, **MySQL 내부 동작 원리 + 실습 기반 분석**을 중심으로 구성되어 있습니다.
    * 개념 → 실습 → 결과 → 결론 구조
    * MySQL 내부 동작(InnoDB, Optimizer) 중심 분석
    * 동시성 환경(session1 / session2) 기반 실험
* 모든 내용은 실제 동작을 기반으로 검증합니다.
<br><br> 

# 폴더 구성
### administration
* MySQL 설치 및 접속- [mysql-installation.md](administration/mysql-installation.md)
* MySQL 설정 관리 - [configuration-management.md](administration/configuration-management.md)
* 데이터베이스 관리 - [database-basic-operations.md](administration/database-basic-operations.md)
* Access Control (계정, 권한, role) - [access-control.md](administration/access-control.md)
* 로그 - [logging.md](administration/logging.md)
* 모니터링 - [monitoring.md](administration/monitoring.md)
* SHOW ENGINE INNODB STATUS 구조 해부 - [show-engine-innodb-status-analysis.md](administration/show-engine-innodb-status-analysis.md)
* 마이그레이션 - [migration.md](administration/migration.md)
#### labs
* MySQL user@host 매칭 우선순위 확인 - [mysql-user-host-matching.md](administration/labs/mysql-user-host-matching.md)
* SQL Security 옵션 (DEFINER / INVOKER) 비교 - [sql-security-option-comparion.md](administration/labs/sql-security-option-comparion.md)
* Slow Query Log 분석 - [slow-query-log-analysis.md](administration/labs/slow-query-log-analysis.md)
* Lock Wait 상황 분석 - [monitoring-routine.md](administration/labs/monitoring-routine.md)
* 스토리지 엔진 별 트랜잭션 지원 여부 비교 - [storage-engine-comparison.md](administration/labs/storage-engine-comparison.md)
* MySQL to Oracle Migration - [mysql-to-oracle-migration.md](administration/labs/mysql-to-oracle-migration.md)
---
### availability
* Backup & Recovery - [backup-and-recovery.md](availability/backup-and-recovery.md)
* Crash Recovery - [crash-recovery.md](availability/crash-recovery.md)
* Binlog & GTID - [binlog-and-GTID.md](availability/binlog-and-GTID.md)
* Replication 및 Clustering - [replication-and-clustering.md](availability/replication-and-clustering.md)
* Failover - [failover.md](availability/failover.md)
#### labs
* mysqldump를 이용한 logical backup & restore - [mysqldump-backup-restore.md](availability/labs/mysqldump-backup-restore.md)
* PITR(Point-in-Time Recovery) 재현 실습 - [PITR.md](availability/labs/PITR.md)
* replication 구축하기 - [replication-setup-and-verification.md](availability/labs/replication-setup-and-verification.md)
* Replication 동작 중 binlog 흐름 확인 - [replication-binlog-flow.md](availability/labs/replication-binlog-flow.md)
* Replication 실패 원인별 트러블슈팅 가이드 - [replication-failure-cases.md](availability/labs/replication-failure-cases.md)
* 수동 failover 실습 - [failover.md](availability/labs/failover.md)
---
### core-concepts
* InnoDB 물리적 저장 구조 - [storage-architecture.md](core-concepts/storage-architecture.md)
* InnoDB 메모리 구조 - [innodb-memory-architecture.md](core-concepts/innodb-memory-architecture.md)
* MySQL SQL Compiler Engine - [sql-compiler-engine.md](core-concepts/sql-compiler-engine.md)
* Metadata 관리 구조 - [metadata-management.md](core-concepts/metadata-management.md)
* 데이터 타입 - [data-types-theory.md](core-concepts/data-types-theory.md)
* 제약조건 - [constraint-theory.md](core-concepts/constraint-theory.md)
* MySQL 내장함수 - [builtin-functions.md](core-concepts/builtin-functions.md)
#### labs
* Buffer Pool Cache 동작 확인 - [buffer-pool-cold-vs-warm-cache-hit-ratio.md](core-concepts/labs/buffer-pool-cold-vs-warm-cache-hit-ratio.md)
* Foreign Key 동작 분석 - [fk-behavior.md](core-concepts/labs/fk-behavior.md)
---
### index
* B+Tree 구조 + 인덱스 - [index-basics.md](index/index-basics.md)
* 인덱스를 타지 않는 쿼리 패턴 - [index-antipatterns.md](index/index-antipatterns.md)
* 인덱스 힌트 - [index-hint.md](index/index-hint.md)
#### labs
* Index Scan 유형 분석 - [explain-index-scan-types.md](index/labs/explain-index-scan-types.md)
* Full Table Scan vs Index Scan 성능 비교 - [full-table-scan-vs-index-scan.md](index/labs/full-table-scan-vs-index-scan.md)
* 옵티마이저의 인덱스 선택 기준 분석 - [index-selection.md](index/labs/index-selection.md)
* 복합 인덱스 leftmost rule 검증 - [composite-index-leftmost-rule.md](index/labs/composite-index-leftmost-rule.md)
* Filesort vs Index Driven Sort 비교 - [filesort-vs-index-ordering.md](index/labs/filesort-vs-index-ordering.md)
* 인덱스 성능 실험1 - [index-performance-experiment.md](index/labs/index-performance-experiment.md)
* 인덱스 성능 실험2 - [index-performance-experiment-2.md](index/labs/index-performance-experiment-2.md)
---
### optimization
* Explain / Explain Analyze - [explain.md](optimization/explain.md)
* 옵티마이저와 통계정보 - [optimizer-statistics.md](optimization/optimizer-statistics.md)
* Optimizer Trace - [optimizer-trace](optimization/optimizer-trace.md)
* Index Condition Pushdown (ICP) - [index-condition-pushdown.md](optimization/index-condition-pushdown.md)
* Limit - [limit.md](optimization/limit.md)
* 암시적 형 변환 - [implicit-type-conversion.md](optimization/implicit-type-conversion.md)
* Join - [join-internal.md](optimization/join-internal.md)
#### labs
* Histogram 유무에 따른 selectivity 추정 변화 확인 - [histogram-selectivity-estimation.md](optimization/labs/histogram-selectivity-estimation.md)
* Stale Stat 상태에서 row estimate 동작 확인 - [stale-statistics-row-estimate.md](optimization/labs/stale-statistics-row-estimate.md)
* Optimizer Trace + Cost Model 분석 - [optimizer-trace-and-cost-model.md](optimization/labs/optimizer-trace-and-cost-model.md)
---
### partitioning
* 파티셔닝 - [partitioning-basics.md](partitioning/partitioning-basics.md)
* Partitioned Index - [partition-index.md](partitioning/partition-index.md)
* 파티션 관리 - [partition-maintenance.md](partitioning/partition-maintenance.md)
#### labs
* Partition Pruning 성능 실험 - [partition-pruning-performance.md](partitioning/labs/partition-pruning-performance.md)
---
### sql-commands
* SQL 명령어 - [sql-command-overview.md](sql-commands/sql-command-overview.md)
* DDL - [sql-ddl.md](sql-commands/sql-ddl.md)
* DML - [sql-dml.md](sql-commands/sql-dml.md)
* DCL - [sql-dcl.md](sql-commands/sql-dcl.md)
* TCL - [sql-tcl.md](sql-commands/sql-tcl.md)
* CTE - [CTE.md](sql-commands/CTE.md)
* 뷰 - [view.md](sql-commands/view.md)
* 트리거 - [trigger.md](sql-commands/trigger.md)
* 저장 프로시저 - [stored-procedure.md](sql-commands/stored-procedure.md)
* 계정 관리 - [account-management.md](sql-commands/account-management.md)
* Role 관리 - [role-management](sql-commands/role-management.md)
---
### transaction
* 트랜잭션 - [transaction-basics.md](transaction/transaction-basics.md)
* 트랜잭션 메커니즘 (undo / redo / MVCC) - [transaction-mechanism.md](transaction/transaction-mechanism.md)
* Consistent Read vs Current Read - [consistent-vs-current-read.md](transaction/consistent-vs-current-read.md)
* 격리 수준 - [isolation-levels.md](transaction/isolation-levels.md)
* Locking (record / gap / next-key lock) - [locking.md](transaction/locking.md)
* MDL - [MDL.md](transaction/MDL.md)
* Deadlock - [deadlock.md](transaction/deadlock.md)
#### labs
* 트랜잭션 lifecycle 확인 - [transaction-implicit-commit.md](transaction/labs/transaction-implicit-commit.md)
* MVCC snapshot 생성 시점 및 동작 확인 - [mvcc-snapshot-check.md](transaction/labs/mvcc-snapshot-check.md)
* Isolation level 별 동작 비교 - [isolation-level-comparison.md](transaction/labs/isolation-level-comparison.md)
* Transaction Anomaly 재현 및 해결 - [transaction-anomaly-experiment.md](transaction/labs/transaction-anomaly-experiment.md)
* Gap Lock 범위 시각화 - [gap-lock-range-visualization.md](transaction/labs/gap-lock-range-visualization.md)
* 인덱스 별 lock 범위 비교 - [locking-index-behavior.md](transaction/labs/locking-index-behavior.md)
* Order by 방식에 따른 lock 범위 비교 - [locking-order-by-behavior.md](transaction/labs/locking-order-by-behavior.md)
* Limit 사용 여부에 따른 lock 범위 비교 - [locking-limit-behavior.md](transaction/labs/locking-limit-behavior.md)