# Metadata 관리 구조
* `metadata`
    * database object를 설명하는 정보
    * table, column, index, constraint, view, routine, privilege 등의 구조 정보를 포함
    * DBMS는 SQL을 실행할 때 metadata를 계속 참조
    * SQL 실행, 권한 검사, optimizer의 실행 계획 생성, DDL 처리 등에 사용
    * [DDL](../sql-commands/sql-ddl.md)은 data 뿐만 아니라 metadata를 변경하는 작업
* `data dictionary`
    * metadata를 저장하고 관리하는 MySQL 내부 저장소
    * 객체 생성 시, data dictionary에 metadata가 등록됨
    * [DDL](../sql-commands/sql-ddl.md) 수행 시 함께 갱신됨
    * MySQL 8.0 이전
        * table metadata가 `.frm` 파일 등에 저장
        * metadata와 storage engine 정보가 분리되어 관리됨
        * crash나 파일 손상 시 metadata 불일치가 발생할 수 있음
    * MySQL 8.0 이후
        * metadata를 InnoDB 기반 transactional data dictionary에서 관리
        * data dictionary가 기존 파일 기반 metadata 저장소를 대체
        * metadata 변경도 트랜잭션적(`transactionally`)으로 처리되어 일관성 향상
* `information_schema`
    * metadata를 조회하기 위한 인터페이스
    * data dictionary의 정보를 사용자에게 제공
## MetaData Lock(MDL)
* metadata의 일관성을 보장하기 위해 사용
* DDL과 DML 사이의 충돌을 방지하는 역할
    * 객체 사용 중 DDL에 의해 metadata가 변경되는 상황 등을 제어
* data dictionary와 별개의 개념이며 MySQL 5.5부터 도입됨
* 자세한 내용은 [transaction 섹션의 MDL 문서](../transaction/MDL.md) 참고