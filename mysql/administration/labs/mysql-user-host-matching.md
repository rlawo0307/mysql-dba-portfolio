# 목차
* 일반 사용자 및 `%` 호스트 계정 생성
* Host 매칭 결과 확인 - **localhost로 접속**
* Host 매칭 결과 확인 - **127.0.0.1 로 접속**
* Host 매칭 결과 확인 - **외부 IP로 접속**
* 요약
<br>

# 일반 사용자 및 `%` 호스트 계정 생성
### 1. 현재 사용자 계정 목록 확인
* `%` 호스트 계정이 있는지,
* 동일한 user가 여러 host로 등록되어 있는지 확인
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
### 2. 일반 사용자 계정 생성
* 정상적인 권장 계정 생성(localhost 또는 특정 IP)
```sql
create user user1@localhost identified by 'aaaa';
```
### 3. `%` 호스트 계정 생성
* 모든 IP에서 접속 가능한 계정 생성
* 미리 생성한 일반 사용자 계정과 username은 동일하지만 host가 다른 `%` 계정 생성
* 동일한 username이라도 host가 다르면 별개의 계정임
* user 또는 host에 특수문자가 포함될 경우 작은따옴표('')로 묶어야함
```sql
create user user2@'%' identified by 'aaaa';
```
### 4. 생성한 계정 확인
```sql
select user, host from mysql.user where user = 'user1';
```
```
+-------+-----------+
| user  | host      |
+-------+-----------+
| user1 | %         |
| user1 | localhost |
+-------+-----------+
```
<br>

# Host 매칭 결과 확인 - `localhost`로 접속
```bash
mysql -u user1 -p -h localhost
```
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

# Host 매칭 결과 확인 - `127.0.0.1`로 접속
* Host 매칭은 IP가 아니라, 서버가 인식한 접속 위치 기준
* 127.0.0.1 IP로 접속하더라도 MySQL은 이를 **루프백 인터페이스**를 통한 로컬 접속으로 인식하며, 해당 접속은 내부적으로 localhost 호스트와 매칭됨
* 루프백 인터페이스(loopback interface)
    * 외부로 나갈 필요 없이 내 컴퓨터가 자기 자신에게 보내는 전용 가상 네트워크 인터페이스
    * IPv4 : 127.0.0.1
    * 호스트명 : localhost

```bash
mysql -u user1 -p -h 127.0.0.1
```
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

# Host 매칭 결과 확인 - 외부 IP로 접속
* 외부 IP로 MySQL에 접속하기 위해 몇 가지 설정 필요
### 1. 서버의 `bind-address` 설정 수정 및 확인
* `/etc/mysql` 내 설정 파일의 모든 bind-address 변수 값을 `0.0.0.0` 또는 `mysql에 접속할 외부 IP`로 설정
    * `0.0.0.0`은 모든 IP를 허용함을 의미
* `mysqlx-bind-address` 는 완전히 다른 서비스이므로 MySQL 접속과 무관함
```bash
grep -R "bind-address" /etc/mysql
```
```
/etc/mysql/mysql.conf.d/mysqld.cnf:bind-address		= 0.0.0.0
/etc/mysql/mysql.conf.d/mysqld.cnf:mysqlx-bind-address	= 127.0.0.1
/etc/mysql/mysql.conf.d/mysql.cnf:bind-address = 0.0.0.0
```
### 2. MySQL 서버 재시작
```bash
sudo systemctl restart mysql
```
### 3. 방화벽(3306) 활성화 및 MySQL 포트(3306) 허용 확인
* 방화벽 상태 확인
```bash
sudo ufw status
```
* 비활성화(inactive) 상태라면 활성화
```bash
sudo ufw enable
```
* MySQL 접속을 허용하는 방화벽 규칙 확인
```bash
sudo ufw status numbered
```
```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere                  
[ 2] 3306/tcp                   ALLOW IN    192.168.0.20  
```
* 3306 포트가 열려있지 않다면 방화벽 규칙 추가
```bash
sudo ufw allow from [MySQL에 접속할 서버 IP] to any port 3306 proto tcp
```
### 4. MySQL 포트 리스닝 상태 확인
* `0.0.0.0:3306` 이 출력될 경우 → success
    * 외부 접속 허용 상태
* `127.0.0.1:3306` 이 출력될 경우 → fail
    * 3306 포트가 아직 루프백만 리슨 중인 경우
```bash
ss -lntp | grep 3306
```
```
LISTEN 0      151          0.0.0.0:3306       0.0.0.0:*          
LISTEN 0      70         127.0.0.1:33060      0.0.0.0:* 
```
### 5. MySQL 안에서 `bind_address` 값 확인 (★반드시 필요★)
* `127.0.0.1`이 출력될 경우 → fail
```sql
show variables like 'bind_address';
```
```
+---------------+---------+
| Variable_name | Value   |
+---------------+---------+
| bind_address  | 0.0.0.0 |
+---------------+---------+
1 row in set (0.00 sec)
```
### 6. 외부 IP로 MySQL 접속 시도
```bash
mysql -u user1 -p -h 192.168.111.129
```
```sql
select user(), current_user();

-- user() : 접속 시도한 사용자와 호스트
-- current_user() : 실제 권한이 적용된 사용자와 호스트
```
```
+-----------------------+----------------+
| user()                | current_user() |
+-----------------------+----------------+
| user1@192.168.111.129 | user1@%        |
+-----------------------+----------------+
1 row in set (0.00 sec)
```
<br>

# 요약
* MySQL 사용자 계정은 `user`와 `host`의 조합으로 구분되며, 동일한 username이라도 host가 다르면 별개의 계정으로 인식된다.
* `localhost` 또는 `127.0.0.1`로 접속하면 MySQL은 이를 로컬 접속으로 인식하여 host가 `host = localhost` 계정과 매칭한다.
* `%` 호스트는 모든 IP를 의미하며 외부 어디서든 접속이 가능하므로, 보안상 매우 위험하다.
* 운영 환경에서는 불필요한 `%` 계정 생성은 피하고, 반드시 필요한 경우에도 접속 가능한 IP를 엄격히 제한하여 보안을 강화해야 한다.
