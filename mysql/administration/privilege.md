# 권한(Privilege)이란?
* MySQL 계정이 수행할 수 있는 작업을 제어하는 권한
* 인증(authentication) 이후 인가(authorization) 과정에서 사용
* [계정](user-account.md)마다 부여 가능
* role에 부여 후 간접적으로 사용할 수도 있음
* **최소 권한 원칙**에 의해 권한 부여
    * 사용자에게 업무 수행에 필요한 최소한의 권한만 부여하는 보안 원칙
    * 불필요한 권한을 제거하고 사고 범위를 최소화
```sql
grant create user on *.* to 'user1'@'localhost';            -- global 권한 (서버 전체)
grant select, insert on db1.* to 'user1'@'localhost';       -- database 권한
grant select on db1.t1 to 'user1'@'localhost';              -- table 권한
grant select(c1, c2) on db1.t1 to 'user1'@'localhost';      -- column 권한
grant execute on procedure db1.proc1 to 'user1'@'localhost'; -- routine 권한 (procedure / function 실행 권한)
```
## 주요 권한 및 범위
* 상위 범위에서 부여된 권한은 하위 범위에 자동 적용됨
    * global → database → table → column
    * routine은 별도의 객체 유형으로 위 계층 구조에 포함되지 않음
    
|                              |의미|global|database|table|column|routine|
|------------------------------|---|:----:|:------:|----:|:----:|:-----:|
|**SELECT<br>INSERT<br>UPDATE**|데이터 조회 / 입력 / 수정   |O|O|O|O|X|
|**DELETE**                    |데이터 삭제                |O|O|O|X|X|
|**CREATE<br>DROP**            |객체 생성 / 삭제           |O|O|X|X|X|
|**ALTER**                     |객체 구조 변경             |O|O|O|X|X|
|**INDEX**                     |인덱스 생성 / 삭제         |O|O|O|X|X|
|**EXECUTE**                   |procedure / function 실행|O|X|X|X|O|
|**CREATE USER**               |사용자 생성               |O|X|X|X|X|
|**GRANT OPTION**              |다른 사용자에게 권한 부여   |O|O|O|O|O|

## 관리 권한
* MySQL 서버 관리 및 운영을 위한 권한
* 모든 관리 권한은 global privilege
|                      |의미|
|----------------------|---|
|**PROCESS**           |전체 세션 및 실행중인 쿼리 조회|
|**RELOAD**            |FLUSH 계열 명령 수행|
|**SHUTDOWN**          |MySQL 서버 종료|
|**REPLICATION SLAVE** |replica가 source의 binary log 읽기|
|**REPLICATION CLIENT**|복제 상태 조회|
|**SUPER**             |과거의 통합 관리자 권한|
## 권한 관리 ([DCL 문서 참고](../sql-commands/sql-dcl.md))