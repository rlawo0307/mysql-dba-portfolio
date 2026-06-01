# 계정 생성하기 (CREATE USER)
* 새로운 MySQL 계정 생성
* 계정 생성 [권한](../administration/privilege.md) 필요
* 계정 생성만 수행하며 권한은 자동으로 부여되지 않음
* 동일한 사용자명이라도 host가 다르면 다른 계정으로 취급
```sql
create user '사용자명'@'host' identified by '비밀번호';
```
### 예시
```sql
create user 'user1'@'localhost' identified by 'aaaa';
create user 'user1'@'127.0.0.1' identified by 'aaaa';
create user 'user1'@'192.168.111.%' identified by 'aaaa';
create user 'user1'@'%' identified by 'aaaa';
create user 'user1' identified by 'aaaa'; -- host 생략 가능. user1@% 계정이 생성됨
```
<br>

# 계정 정보 변경하기 (ALTER USER)
* 기존 MySQL 계정의 속성 변경
* 비밀번호, 비밀번호 만료 정책, 인증 플러그인, 계정 잠금 상태 등을 변경할 수 있음
### 비밀번호 변경
```sql
alter user '사용자명'@'host' identified by '새 비밀번호';
```
### 비밀번호 만료 정책 변경
```sql
alter user '사용자명'@'host' password expire; -- 즉시 만료
alter user '사용자명'@'host' password expire interval N day; -- 일정 기간 후 만료
alter user '사용자명'@'host' password expire never; -- 만료 없음
alter user '사용자명'@'host' password expire default; -- 기본 정책 사용
```
### 인증 플러그인 변경
* 플러그인에 따라 비밀번호가 필요할 수 있음
```sql
alter user '사용자명'@'host' identified with 플러그인 [by '비밀번호'];
```
### 계정 잠금 상태 변경
```sql
alter user '사용자명'@'host' account lock; -- 계정 잠금
alter user '사용자명'@'host' account unlock; -- 계정 잠금 해제
```
<br>

# 계정 이름 변경하기 (RENAME USER)
* 사용자명 또는 host 변경
* 권한 정보는 유지됨
```sql
rename user '기존 사용자명'@'기존 host' to '변경할 사용자명'@'변경할 host';
```
<br>

# 계정 삭제하기 (DROP USER)
* 권한이 있는 계정에서 삭제 가능
* 계정에 부여된 권한도 함께 제거됨
* 여러 계정를 한번에 삭제 가능
```sql
drop user '사용자명'@'host';
```
### 예시
```sql
drop user 'user1'@'localhost';
drop user 'user1'@'127.0.0.1';
drop user 'user1'@'192.168.111.%';
drop user 'user1'@'%';
drop user 'user1'; -- host 생략 가능. user1@% 계정이 삭제됨
```
<br>

# 계정 조회하기
* mysql.user 조회 권한 필요
* mysql.user 테이블에 저장된 계정 메타데이터 조회
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
| root             | localhost | -- 관리자 계정
| user1            | localhost | -- 사용자 계정
+------------------+-----------+
6 rows in set (0.00 sec)
```
* MySQL 또는 OS 패키지가 생성한 시스템 계정도 함께 조회될 수 있음
    * debian-sys-maint, mysql.infoschema, mysql.session, mysql.sys 등
<br>

# 접속 및 인증 계정 확인
* `user()`
    * 클라이언트 접속 시 사용한 계정
* `current_user()`
    * MySQL이 host matching 후 실제 인증에 사용한 계정
    * 권한 적용 기준
```sql
select user(), current_user();
```
* 접속 계정 = 인증 계정 → 접속 요청한 계정으로 인증
* 접속 계정 ≠ 인증 계정 → host matching 결과 다른 계정으로 인증