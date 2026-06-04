# MySQL 설정 관리
## my.cnf란?
* MySQL/MariaDB 계열에서 사용하는 설정 파일
* 어원
    * `my` = 역사적으로 MySQL에서 사용하던 prefix
    * `cnf` = configuration
## 설정 적용 방식
* startup
    * 프로그램 시작 시 설정 파일을 읽고 적용
    * 파일 수정 후 재시작 필요
    * runtime 변경이 불가능한 read-only 변수 존재
* runtime
    * 동적 변수는 실행 중 변경 가능
    * 변경 범위는 session 도는 global
## 설정 파일 위치 및 내용 확인
### 설정 파일 경로 확인
```bash
mysqld --verbose --help | grep -A 1 "Default options"
```
```bash
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
```
* `/etc/my.cnf` → `/etc/mysql/my.cnf` → `~/.my.cnf` 순서대로 읽음
* 여러 파일이 있으면 뒤에 있는 파일이 앞 설정을 덮어씀
    * 같은 옵션일 때만 override됨
    * 서로 다른 옵션은 누적 적용됨
### 설정 파일이 존재하는지 확인
* MySQL은 여러 설정 파일을 순차적으로 탐색
* 모든 설정 파일이 존재할 필요는 없음
* 실제 환경에서는 여러 설정 파일 중 일부만 존재하는 경우가 일반적
* 파일이 없다면 무시하고 그냥 넘어감
```bash
for f in /etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
do
    if [ -f "$f" ]; then
        echo "[O] $f"
    else
        echo "[X] $f"
    fi
done
```
```bash
[X] /etc/my.cnf
[O] /etc/mysql/my.cnf       # 현재 `/etc/mysql/my.cnf` 파일만 존재
[X] /home/dbadmin/.my.cnf
```
### 설정 파일 내용 확인
* `my.cnf`는 설정을 직접 담고 있지 않으며 다른 설정 파일들을 include 하고 있는 파일임
* include된 파일은 `my.cnf`보다 나중에 읽힘
* include된 디렉토리는 선언된 순서대로 읽힘
    * 디렉토리 내 파일은 파일명 기준 알파벳 순으로 읽힘
* 뒤에 있는 파일이 override 가능
```bash
cat /etc/mysql/my.cnf
```
```bash
!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/

# `my.cnf` → `conf.d` → `mysql.conf.d`
# 실제 적용되는 설정은 include된 파일까지 모두 고려해야 함
```
* `/etc/mysql/conf.d/`
    * 추가 설정 파일을 두는 디렉토리
    * 해당 디렉토리 하위의 *.cnf 파일 전부를 읽음
* `/etc/mysql/mysql.conf.d/`
    * Ubuntu 패키지 기본 mysqld 설정 파일이 위치하는 디렉토리
    * 해당 디렉토리 하위의 *.cnf 파일 전부를 읽음
## 주요 섹션
### [mysqld]
* MySQL Server 프로세스(mysqld) 설정
* 대표 설정
    * 클라이언트 연결 관련
        * `max_connections` : 동시 접속 가능한 최대 connection 수
        * `connect_timeout` : 클라이언트 연결 요청 후 handshake 완료까지 대기 시간
        * `wait_timeout` : 비대화형 idle connection 종료 시간
        * `interactive_timeout` : 대화형 idle connection 종료 시간
        * `max_connect_errors` : host 별 연속 연결 실패 허용 횟수
    * 네트워크 관련
        * `port` : MySQL 서버가 연결 요청을 기다리고 있는 포트 번호 (default = 3306)
        * `bind-address` : MySQL 서버가 어떤 IP 주소에서 연결 요청을 받을지
            * 127.0.0.1 : 로컬만 허용
            * 0.0.0.0 : 외부 접속 허용 
        * `socket` : 로컬에서 MySQL client와 server가 통신할 때 사용하는 Unix domain socket 파일 경로
        * `skip-name-resolve` : DNS reverse lookup 비활성화
### [client]
* 모든 client 프로그램 공통
* 읽는 대상 : mysql client, mysqldump, mysqladmin, mysqlcheck, mysqlbinlog, 기타 client tools
* 대표 설정
    * `user` : 접속할 MySQL server 계정
    * `password` : 접속할 MySQL server 비밀번호
        * 파일에는 암호화 없이 평문으로 저장되므로 보안상 주의!
    * `port` : 접속할 MySQL server 포트 번호
    * `socket` : 로컬 접속 시 사용할 Unix domain socket 파일 경로
        * Unix socket 기반 로컬 접속을 사용할 경우, [client]와 [mysqld]의 socket 경로가 동일해야 함
        * TCP 접속이면 상관 없음
    * `default-character-set` : client 기본 문자셋
### [mysql]
* MySQL CLI 전용
* [client] 섹션과 중복된 내용이 있을 경우, [mysql]을 우선으로 함
* 대표 설정
    * `prompt` : MySQL CLI prompt 형식
    * `pager` : 조회 결과 출력 시 사용할 pager 프로그램
    * `default-character-set` : MySQL CLI 기본 문자셋
### [mysqldump]
* mysqldump 전용
* 대표 설정
    * `quick` : row 단위로 읽어 메모리 사용량을 줄이며 dump 수행
    * `single-transaction` : consistent read 기반으로 snapshot dump 수행
    * `max_allowed_packet` : dump 중 처리 가능한 최대 packet 크기
## 시스템 변수 조회 및 변경
* 설정 파일에 정의된 값은 서버 시작 시 시스템 변수로 적용됨
* 실제 적용된 설정 값은 설정 파일이 아닌 `show variables` 결과로 확인해야 함
    * 동적으로 변경된 값이 있을 수 있음
### 전체 변수 조회
```sql
show global variables;  -- 서버 전체 변수 조회
show session variables; -- 세션 변수 조회, show variables; 와 동일
```
### 특정 변수 조회
```sql
show global variables like 'max_connections';
```
```sql
+-----------------+-------+
| Variable_name   | Value |
+-----------------+-------+
| max_connections | 151   |
+-----------------+-------+
1 row in set (0.00 sec)
```
### 변수 설정 변경하기
* 세션 단위 변경
```sql
set session sql_mode = 'STRICT_TRANS_TABLES';
```
* 글로벌 단위 변경
```sql
set global autocommit = 0;
```
* 변경 실패
```sql
set session innodb_buffer_pool_load_at_startup = off; -- fail
set global innodb_buffer_pool_load_at_startup = off; -- fail
```
```sql
ERROR 1238 (HY000): Variable 'innodb_buffer_pool_load_at_startup' is a read only variable
-- read-only 변수는 실행 중 변경할 수 없음
-- 설정 파일에서 수정 후 서버 재시작 필요
```