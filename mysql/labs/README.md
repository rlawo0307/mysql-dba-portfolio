# MySQL 실습 (MySQL Labs)
이 디렉터리는 MySQL DBA 관점에서 운영 환경에 적용하기 전 반드시 검증해야 할 설정과 동작을 실제 환경에서 실험하고 기록합니다.

## 목적
* MySQL 운영 환경 적용 전 사전 검증 수행

## 범위
이 디렉터리의 실습은 다음을 포함합니다.
* MySQL 사용자 계정과 호스트 매칭 이해 실습 - [mysql-user-host-matching.md](mysql-user-host-matching.md)
* MySQL 쿼리 실행 계획 이해 (기본) - [explain-table-access-types.md](explain-table-access-types.md)

## 주의 사항
본 실습에는 보안상 위험한 설정이 일부 포함되어 있으며, **운영 환경에서는 절대 그대로 적용해서는 안 됩니다.**

모든 실습은 로컬 VM 기반 테스트 환경에서만 수행되었습니다.