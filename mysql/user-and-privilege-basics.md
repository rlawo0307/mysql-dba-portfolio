# 계정 조회하기
* MySQL은 접속할 때 정확히 일치하는 계정이 없으면 명단(MySQL 계정 목록)에서 가장 비슷한 계정을 찾아서 씀
    * user(), current_user()가 같다면, 단일 계정 매칭
    * 다르다면, 대체 계정으로 권한 적용 중
### MySQL 사용자 계정 목록 조회
* 관리자 권한 필요
* 일반 사용자는 조회할 수 없음
```sql
select user, host from mysql.user;
```
```
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| debian-sys-maint | localhost |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+
5 rows in set (0.00 sec)
```
* debian-sys-maint@localhost
    * Debian/Ubuntu에서 MySQL 서비스 관리용
    * 패키지 업데이트나 서비스 재시작 시 사용
    * 지우면 안됨
    * (로그인은 가능하지만) 일반 사용자가 직접 사용하는 계정은 아님
* mysql.infoschema@localhost
    * information_schema 내부용
    * 로그인 불가
* mysql.session@localhost
    * MySQL 내부 세션용 계정
    * 로그인 불가
* mysql.sys@localhost
    * sys 데이터베이스 뷰의 DEFINER
    * 성능/상태 조회용
    * 로그인 불가
* root@localhost
    * 관리자 계정
### 접속 시도할 때 사용한 계정 조회
```sql
select user();
```
```
+----------------+
| user()         |
+----------------+
| root@localhost |
+----------------+
```
### 실제 권한이 적용된 계정 조회
```sql
select current_user();
```
```
+----------------+
| current_user() |
+----------------+
| root@localhost |
+----------------+
```
<br>

# 계정 생성하기
* 권한이 있는 계정에서 생성 가능
* 계정 = (user + host)
* host 자리에 올 수 있는 것
    * localhost
        * 같은 서버에서만 접속 가능
        * 보통 소켓 접속
        * 실무 기본 값
    * IP 주소
        * 특정 IP에서만 접속 허용
        * localhost와 127.0.0.1은 다른 host
    * IP 대역 (와일드카드)
        * 내부망 접속 허용
    * 도메인 이름
    * % (모든 곳)
        * host name 생략 시 자동으로 기본값으로 부여
        * __위험한 계정 생성 방식으로, 실무에서는 host를 생략한 계정 생성은 거의 금지 수준이며 반드시 localhost 또는 명시적인 IP를 사용해야 함__
* password를 생략하면 계정은 생성되지만 인증 불가 상태로 로그인 할 수 없음
### 계정 생성
```sql
create user user1@localhost identified by 'aaaa';
```
* user name : user1
* host name : localhost
* password : aaaa
### host를 생략하여 계정 생성
```sql
create user user1 identified by aaaa;
```
* user name : user1
* host name : %
* password : aaaa
### 생성된 계정 확인
```sql
select user, host from mysql.user;
```
```
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| debian-sys-maint | localhost |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
| user1            | localhost |
+------------------+-----------+
6 rows in set (0.00 sec)
```
<br>

# 계정 전환하기
* MySQL 내부에서 계정을 전환하는 방법은 없음
* MySQL을 종료하고 새로운 계정으로 접속해야 함
### mySQL 종료
```sql
exit;
```
### 특정 계정으로 접속
```bash
mysql -u user1 -p
```
### 접속한 계정 확인
```sql
select user(), current_user();
```
```
+-----------------+-----------------+
| user()          | current_user()  |
+-----------------+-----------------+
| user1@localhost | user1@localhost |
+-----------------+-----------------+
1 row in set (0.00 sec)
```
<br>

# 계정 삭제하기
* 권한이 있는 계정에서 삭제 가능
### 계정 삭제
```sql
drop user user1@localhost;
```
### host를 생략하여 계정 삭제
* user1@% 의 계정이 삭제됨
```sql
drop user user1;
```
<br>

# 권한 부여하기 (GRANT)
### root 계정 권한 조회
```sql
show grants for 'root'@'localhost';
```
```
GRANT PROXY ON ``@`` TO `root`@`localhost` WITH GRANT OPTION
```
* root@localhost는 모든 사용자로 가장할 수 있고 그 권한을 남에게도 줄 수 있다.
    * GRANT PROXY
        * PROXY 권한을 부여한다
        * 다른 사용자로 가장할 수 있는 권한
    * ON \`\`@\`\`
        * 모든 사용자, 모든 호스트
        * PROXY 전용 전체 지정자 ('*' 과 같은 의미)
    * TO \`root\`@\`localhost\`
        * 권한을 받는 계정
    * WITH GRANT OPTION
        * 다른 계정에도 권한을 부여할 수 있음
### user1 계정 권한 조회 (BEFORE)
```sql
show grants for user1@localhost;
```
```
+-------------------------------------------+
| Grants for user1@localhost                |
+-------------------------------------------+
| GRANT USAGE ON *.* TO `user1`@`localhost` |
+-------------------------------------------+
1 row in set (0.00 sec)
```
* user1@localhost는 서버에 로그인만 가능하고, 어떤 DB 접근 권한이 없다.
    * GRANT USAGE
        * 권한 없음
        * 최소 권한
        * 아무것도 허용하지 않음
    * ON \*.\*
        * 모든 DB, 모든 테이블
### 권한 부여
* MySQL 권한 부여는 범위를 좁게, 권한을 최소로 주는 것을 원칙으로 한다.
* 권한 부여는 권한을 줄 수 있는 권한을 가진 계정(관리자 계정)에서 가능
    * GRANT OPTION 권한 필요
    * root 계정은 모든 권한을 가지고 있으므로 가능
```sql
grant all privileges on db1.* to user1@localhost;
```
* 데이터베이스 db1에 대한 모든 권한을 부여
* 단, 해당 권한을 다른 계정에 부여할 수 없음
    * WITH GRANT OPTION 없음
    * REVOKE도 불가
### 권한 부여 후 확인 (AFTER)
```sql
show grants for user1@localhost;
```
```
+--------------------------------------------------------+
| Grants for user1@localhost                             |
+--------------------------------------------------------+
| GRANT USAGE ON *.* TO `user1`@`localhost`              |
| GRANT ALL PRIVILEGES ON `db1`.* TO `user1`@`localhost` |
+--------------------------------------------------------+
2 rows in set (0.00 sec)
```
* user1으로 접속 후 권한을 받은 db에 접근할 수 있어야 함
```sql
show databases;
```
```
+--------------------+
| Database           |
+--------------------+
| db1                |
| information_schema |
| performance_schema |
+--------------------+
3 rows in set (0.00 sec)
```
```sql
use db1; -- success!
```
```sql
use mysql; -- fail (권한 없음, 보안 정상)
```
<br>

# 여러가지 권한 부여 방법
### 읽기 전용 계정 (조회용)
* select만 가능
* 그 외 다른 DML 불가능
```sql
grant select on db1.* to user1@localhost;
```
### table 단위 권한
* db1의 t1 테이블에만 접근 가능
```sql
grant select, insert on db1.t1 to user1@localhost;
```
### 컬럼 단위 권한
* db1의 t1 테이블의 c1, c2 컬럼만 조회 가능
```sql
grant select (c1, c2) on db1.t1 to user1@localhost;
```
### table 관리 권한
```sql
grant create, alter, drop on db1.* to user1@localhost;
```
### 모든 DB 조회 권한
```sql
grant select on *.* to user1@localhost;
```
### DB 생성, 삭제 권한
```sql
grant create, drop on *.* to user1@localhost;
```
### PROXY 권한
* admin_user가 user1처럼 행동 가능
* 일반적인 서비스 운영에서는 거의 사용하지 않는 고급 권한
```sql
grant PROXY on user1@localhost to admin_user@localhost;
```
<br>

# 권한 삭제하기 (REVOKE)
* 권한을 회수할 수 있는 계정
    * 해당 권한을 가지고 있고,
    * 그 권한에 대해 grant option이 있는 계정
    * root / SUPER 계정
    * __권한을 부여한 계정만 권한을 회수할 수 있는 것은 아니다.__
### 권한 회수
```sql
revoke all privileges on db1.* from user1@localhost;
```
### 권한 회수 후 확인
```sql
show grants for user1@localhost;
```
```
+-------------------------------------------+
| Grants for user1@localhost                |
+-------------------------------------------+
| GRANT USAGE ON *.* TO `user1`@`localhost` |
+-------------------------------------------+
1 row in set (0.00 sec)
```