# MySQL 계정(Account)이란?
* MySQL Server에 접속하기 위한 사용자 정보
* MySQL은 계정을 기준으로 인증과 권한 부여를 수행
* `(user + host)` 조합으로 식별
    * user : 사용자 이름
    * host : 접속 가능한 클라이언트 위치
    * `'user1'@'localhost'`, `'user1'@'192.168.0.%'`, `'user1'@'%'` 모두 다른 계정임
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
* 클라이언트가 MySQL에 접속 시, MySQL은 이 접속 요청을 `mysql.user` 테이블의 어떤 계정으로 인증할 것인가를 판단
* 접속 요청이 들어오면 존재하는 계정 중 조건을 만족하는 계정을 찾음
* 여러 계정이 동시에 조건을 만족하면 더 구체적인 host를 가진 계정을 우선 선택
```text
ex) 접속 요청 계정 : 'user1'@'192.168.111.130'

[mysql.user]
'user1'@'%'
'user1'@'192.168.111.%'
'user1'@'192.168.111.130'

→ mysql.user에 존재하는 세 계정 모두 접속 조건을 만족
→ 더 구체적으로 일치하는 'user1'@'192.168.111.130'을 선택
```
### host matching 결과 확인
```sql
select user(), current_user();
-- user()           : 클라이언트 접속 시 사용한 계정
-- current_user()   : MySQL이 host matching 후 실제 인증에 사용한 계정
```
* user()와 current_user()가 같음 → 단일 계정 매칭
* user()와 current_user()가 다름 → 대체 계정으로 권한 적용 중
* **권한은 current_user() 기준으로 적용됨**
<br><br>

# MySQL 계정 관리
## 계정 생성
* 권한이 있는 계정에서 생성 가능
```sql
create user 'user1'@'localhost' identified by 'aaaa';
create user 'user1'@'127.0.0.1' identified by 'aaaa';
create user 'user1'@'192.168.111.%' identified by 'aaaa';
create user 'user1' identified by 'aaaa'; -- host 생략 가능. user1@% 계정이 생성됨
```
## 계정 조회
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
| user1            | localhost |
+------------------+-----------+
6 rows in set (0.00 sec)
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
* `user1@localhost`
    * 사용자가 생성한 계정
## 계정 전환
* MySQL 내부에서 계정을 전환하는 방법은 없음
* MySQL을 종료하고 새로운 계정으로 접속해야 함
```sql
exit; -- mysql 종료 (sql)
```
```bash
mysql -u user1 -p # 특정 계정으로 접속 (bash)
```
## 계정 삭제
* 권한이 있는 계정에서 삭제 가능
```sql
drop user 'user1'@'localhost';
drop user 'user1'@'127.0.0.1';
drop user 'user1'@'192.168.111.%';
drop user 'user1'; -- user1@% 계정이 삭제. 해당 계정이 없으면 에러 출력
```