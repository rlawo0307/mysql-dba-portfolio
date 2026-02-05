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
```sql
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
* `debian-sys-maint@localhost`
    * Debian/Ubuntu에서 MySQL 서비스 관리용
    * 패키지 업데이트나 서비스 재시작 시 사용
    * 지우면 안됨
    * (로그인은 가능하지만) 일반 사용자가 직접 사용하는 계정은 아님
* `mysql.infoschema@localhost`
    * information_schema 내부용
    * 로그인 불가
* `mysql.session@localhost`
    * MySQL 내부 세션용 계정
    * 로그인 불가
* `mysql.sys@localhost`
    * sys 데이터베이스 뷰의 DEFINER
    * 성능/상태 조회용
    * 로그인 불가
* `root@localhost`
    * 관리자 계정
### 접속 시도할 때 사용한 계정 조회
```sql
select user();
```
```sql
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
```sql
+----------------+
| current_user() |
+----------------+
| root@localhost |
+----------------+
```
<br>

# 계정 생성하기
* 권한이 있는 계정에서 생성 가능
* 계정 = `(user + host)`
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
```sql
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
```sql
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