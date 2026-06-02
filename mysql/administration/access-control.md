# Access Control
* MySQL은 계정(account)을 기준으로 인증(authentication) 수행
    * `인증` : 사용자의 신원을 확인하는 과정 
* 인증된 계정의 권한(privilege)과 역할(role)을 기준으로 인가(authorization) 수행
    * `인가` : 사용자가 수행 가능한 작업을 판단하는 과정
<br><br>

# 계정 (Account)
* MySQL Server에 접속하기 위한 사용자 정보
* `(user + host)` 조합으로 식별
    * user : 사용자 이름
    * host : 접속 가능한 클라이언트 위치
* 계정 단위로 인증 및 권한 부여를 수행
* 계정 상태를 관리하기 위한 기능 제공
    * `account lock`
        * 특정 계정을 로그인 불가능 상태로 만드는 기능
        * 계정 자체는 유지되지만 인증 단계에서 차단
    * `password expire`
        * 계정의 비밀번호를 강제로 변경하게 만드는 기능
        * 로그인은 가능하지만 비밀번호 변경 전까지 정상 사용 불가
* 계정마다 인증 방식이 다를 수 있음
    * `authentication plugin`(인증 플러그인)
        * 비밀번호를 어떤 방식으로 검증할지를 결정
        * caching_sha2_password, mysql_native_password, sha256_password 등
        * MySQL 8.0은 `caching_sha2_password`를 기본으로 사용
## host 종류
* localhost
    * 로컬 서버에서만 접속 가능
    * 보통 Unix Socket을 통해 접속
    * 127.0.0.1 과는 다른 host로 취급
* 특정 IP 주소
    * 해당 IP에서만 접속 가능
* 대역 지정
    * 내부망 접속 허용 
* 모든 호스트
    * 모든 IP 허용
    * host name 생략 시 '%'가 기본값으로 사용됨
    * 위험한 계정 생성 방식이므로 운영 환경에서는 주의 필요 
## MySQL host matching
* 클라이언트가 접속하면 MySQL은 `mysql.user` 테이블에서 인증에 사용할 계정을 찾음
* 접속 요청이 들어오면 존재하는 계정 중 조건을 만족하는 계정을 찾음
* 여러 계정이 동시에 조건을 만족하면 더 구체적인 host를 가진 계정을 우선 선택
* host matching 완료 후 선택된 계정의 인증 플러그인을 사용하여 인증 수행
```text
ex) 접속 요청 계정 : 'user1'@'192.168.111.130'

[mysql.user]
'user1'@'%'
'user1'@'192.168.111.%'
'user1'@'192.168.111.130'

→ mysql.user에 존재하는 세 계정 모두 접속 조건을 만족
→ 더 구체적으로 일치하는 'user1'@'192.168.111.130'을 선택
```
#### ※ 계정 관리 문법 및 사용은 [account-management](../sql-commands/account-management.md) 문서 참고
<br>

# 권한 (Privilege)
* MySQL 계정이 수행할 수 있는 작업을 제어하는 권한
* 계정마다 부여 가능
* role에 부여 후 간접적으로 사용할 수도 있음
* **최소 권한 원칙**에 의해 권한 부여
    * 사용자에게 업무 수행에 필요한 최소한의 권한만 부여하는 보안 원칙
    * 불필요한 권한을 제거하고 사고 범위를 최소화
* 범위
    * `global` 권한 (서버 전체)
    * `database` 권한
    * `table` 권한
    * `column` 권한
    * `routine` 권한 (procedure / function 실행 권한)
* 상위 범위에서 부여된 권한은 하위 범위에 자동 적용됨
    * global → database → table → column
    * routine은 별도의 객체 유형으로 위 계층 구조에 포함되지 않음
## 주요 권한    
|                              |의미|global|database|table|column|routine|
|------------------------------|---|:--:|:------:|:---:|:----:|:-----:|
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
|**REPLICATION REPLICA** |replica가 source의 binary log 읽기|
|**REPLICATION CLIENT**|복제 상태 조회|
|**SUPER**             |과거의 통합 관리자 권한<br>(현재는 여러 dynamic privilege로 분리됨)|
#### ※ 권한 관리 문법 및 사용은 [sql-dcl](../sql-commands/sql-dcl.md) 문서 참고
<br>

# 역할 (Role)
* 여러 권한을 하나의 묶음으로 관리하기 위한 객체
* MySQL 8.0 부터 지원
* 사용자에게 직접 권한을 부여하는 대신 role에 권한을 부여한 후 사용자에게 할당 가능
* 다수 사용자의 권한 관리를 단순화
    * 하나의 role에 여러 권한 부여 가능
    * 하나의 사용자에게 여러 role 부여 가능
    * role끼리 계층적으로 부여 가능
    * 활성화된 role의 권한만 사용 가능
* 권한 변경 시 사용자별 권한을 수정하는 대신 role만 수정하면 됨
* role을 부여받아도 자동으로 활성화되는 것은 아님
    * 로그인 시 자동 활성화하려면 `default role` 설정 필요
```text
read_only_role
 └─ select

developer_role
 ├─ read_only_role
 ├─ insert
 ├─ update
 └─ delete

schema_manager_role
 ├─ create
 ├─ alter
 └─ drop

dba_role
 ├─ developer_role
 ├─ create user
 ├─ process
 └─ reload
```
```text
user1
 ├─ developer_role      → select, insert, update, delete
 └─ schema_manager_role → create, alter, drop 

user2
 ├─ read_only_role      → select
 └─ schema_manager_role → create, alter, drop 

user3
 └─ dba_role → select, insert, update, delete,
             → create user, process, reload
 ```
#### ※ role 관리 문법 및 사용은 [role-management](../sql-commands/role-management) 문서 참고