# Migration이란?
* 기존 database 환경의 schema, data, object, 권한, application 의존 요소를 다른 환경으로 이전하는 작업
* 단순히 data를 복사하는 작업이 아님
* source DB와 target DB 사이의 차이를 분석, 변환, 검증하는 과정까지 포함
* 핵심은 단순 이관이 아니라 target DB 환경에서 정상적으로 동작하는지 검증하는 것
## Migration 대상
* schema
    * database
    * table
    * column
    * constraint
    * index
* data
    * row data
    * large object data
* object    
    * view
    * procedure
    * function
    * trigger
    * event / job
* 권한
    * user
    * role
    * privilege
* application 의존 요소
    * SQL 문법
    * pagination query
    * 날짜 함수
    * auto increment / sequence 처리
    * transaction 처리 방식
## Migration 종류
* 동일 DBMS 간 migration
    * 같은 DBMS 안에서 database를 옮기는 작업
    * 서버 이전
    * version upgrade
    * on-premise → cloud 이전
    * 개발 환경 → 운영 환경 이전
* 이기종 DBMS 간 migration
    * 서로 다른 DBMS 사이에서 database를 옮기는 작업 
    * DBMS마다 구조와 문법이 다르기 때문에 변환 작업이 필요
    * data type, SQL 문법, index, constraint, sequence, function 차이를 반드시 검토해야 함
## Migration 방식 
* `logical migration`
    * SQL dump, export / import, ETL 도구 등을 사용해 논리적으로 데이터를 이전하는 방식
    * 일반적으로 dump / import 방식이 많이 사용됨
    * schema와 data를 SQL 또는 export 파일 형태로 추출한 뒤 target DB에 적재
    * 장점
        * 이기종 DBMS 간 migration 가능
        * 필요한 table만 선택적으로 이관 가능
        * schema 변환 과정을 확인하기 쉬움
    * 단점
        * 대용량에서는 시간이 오래 걸릴 수 있음
        * 이관 중 source DB 변경분에 대한 처리가 필요할 수 있음
        * object 변환이 완전하지 않을 수 있음
* `physical migration`
    * data file, backup image, storage snapshot 등을 이용해 물리적으로 이전하는 방식
    * 일반적으로 동일 DBMS 또는 동일 계열 환경에서 사용
    * 장점
        * 대용량 데이터 이전에 유리할 수 있음
        * logical migration보다 속도가 빠를 수 있음
    * 단점
        * 이기종 DBMS 간 migration에는 적합하지 않음
        * DB version, OS, storage 구조 의존성이 있음
## Migration 기본 절차
```text
1. source DB 구조 분석
2. source DB - target DB 호환성 분석
3. schema 변환
    - source DB의 DDL을 target DB에 맞게 변환
4. data 이관
    - source DB의 데이터를 target DB에 적재
    - 방식은 도구나 환경에 따라 다름
5. 검증
    - migration은 이관보다 검증이 더 중요
    - row count 일치 여부
    - PK / FK / unique constraint 유지 여부
    - index 생성 여부
    - null 값 유지 여부
    - 날짜 값 변환 여부
    - 문자 깨짐 여부
    - 숫자 정밀도 손실 여부
    - query 성능 차이 여부 등
6. application 호환성 검토
    - DBMS를 바꾸면 application SQL도 영향을 받을 수 있음
7. cutover
    - 실제 운영 DB를 target DB로 전환하는 단계
    - downtime이 발생할 수 있으므로 사전 계획 필요
    - 마지막 변경분 동기화 방법도 함께 고려해야 함
8. rollback 계획
    - migration 실패 시 원래 source DB로 되돌리는 계획
    - 운영 migration에서는 필수
```