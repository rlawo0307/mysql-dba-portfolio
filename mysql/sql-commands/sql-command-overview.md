# SQL 명령어란?
* SQL = Structured Query Language
* 데이터베이스와 소통하기 위한 표준 언어
* 역할에 따라 4가지 그룹으로 나뉘며, 데이터베이스를 다루는 서로 다른 작업을 담당
    * `DDL` : 데이터베이스 구조 정의
    * `DML` : 데이터 조작
    * `DCL` : 권한과 보안 처리
    * `TCL` : 트랜잭션 제어
<br>

# DDL
* DDL = Data Definition Language
* 데이터베이스의 구조를 정의하고 변경하는 명령어
* 주요 명령어 : `CREATE`, `ALTER`, `DROP`, `TRUNCATE`
* [→ DDL에 대한 내용 자세히 보기](sql-ddl.md)
# DML
* DML = Data Manipulation Language
* 데이터베이스 내에 저장된 데이터를 조작하는 명령어
* 주요 명령어 : `SELECT`, `INSERT`, `UPDATE`, `DELETE`
* [→ DML에 대한 내용 자세히 보기](sql-dml.md)
### CRUD란?
* 데이터베이스나 애플리케이션에서 데이터를 다루는 기본적인 4가지 작업
* 주요 작업 : `Create`, `Read`, `Update`, `Delete`
* **`CRUD`와 `DML`의 연관성**
    * CRUD = 개념적인 데이터 조작 작업의 분류
    * DML = CRUD 작업을 수행하는 실제 SQL 명령어
# DCL
# TCL