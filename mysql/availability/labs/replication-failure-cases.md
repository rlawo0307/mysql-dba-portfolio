# Replication 실패 원인별 트러블슈팅 가이드
## 1. server_id가 동일한 경우
```sql
Source_Host             : localhost
Replica_IO_Running      : NO -- source 연결 실패
Replica_SQL_Running     : Yes -- 기존 relay log가 있다면 SQL thread는 계속 동작 가능
Seconds_Behind_Source   : 0
Last_IO_Error           : source and replica have equal MySQL server ids
Last_SQL_Error          : 
```
### 현상
* source와 replica의 server_id가 같아서 replica I/O thread가 source에 접속하자마자 즉시 거부됨
### 원인
* source와 replica의 server_id가 같음
* replication에서는 각 서버를 server_id로 식별하므로 반드시 고유해야 함
### 해결
* source와 replica 각 서버의 .cnf에서 server_id를 서로 다르게 설정하기
* 같은 머신의 서로 다른 MySQL 인스턴스라면 localhost 또는 127.0.0.1 사용 가능
* 서로 다른 머신이라면 localhost 사용 불가. 실제 source IP 또는 % 사용하기
* 설정 후 MySQL, replication 재시작 필요
<br>

## 2. source 서버 연결 실패
```sql
Source_Host             : 192.168.111.129
Replica_IO_Running      : Connecting -- source에 접속을 계속 재시도 중
Replica_SQL_Running     : Yes -- 기존 relay log가 있다면 SQL thread는 계속 동작 가능
Seconds_Behind_Source   : 0
Last_IO_Error           : Error connecting to source 'repl@192.168.111.129:3306'. This was attempt 3/86400, with a delay of 60 seconds between attempts. Message: Can't connect to MySQL server on '192.168.111.129:3306' (110)
Last_SQL_Error          : 
```
### 현상
* replica가 source 서버에 TCP 연결을 시도했지만 실패
* 에러 코드(110)은 connection timeout으로, 네트워크 레벨에서 연결 자체가 안되는 상태
* MySQL 내부 로직 오류가 아니라 TCP 연결 단계에서 발생한 네트워크 문제
### 원인
* source MySQL 서버가 실행 중 아닌 경우
* 잘못된 IP 또는 포트를 설정한 경우
* 방화벽/보안 그룹에서 3306 포트를 차단한 경우
* bind-address 설정으로 외부 접속이 제한된 경우
* 네트워크 자체가 단절된 경우(ping 불가 등)
### 해결
* [source] - source 서버 상태가 `active`인지 확인
```bash
systemctl status mysql
```
* [source, replica] - IP 확인 후 source 접속 방법 확인
```bash
hostname -I # source에서 수행
```
```bash
192.168.111.129
```
```sql
-- replica에서 수행
change replication source to
    source_user='binlog_reader',
    source_host='192.168.111.129',  -- 접속할 source IP. 동일한지 확인
    source_port=3306,
    source_password='aaaa',
    source_auto_position=1,
    get_source_public_key=1;

-- 설정 후 replication 재시작 필요
```
* [source] - 방화벽/포트 오픈 확인
```bash
sudo ufw allow 3306 # 3306 포트 접속 허용
```
```bash
sudo ufw status
```
```bash
Status: active

To                         Action      From
--                         ------      ----z
22/tcp                     ALLOW       Anywhere                  
3306/tcp                   ALLOW       192.168.0.20              
3306                       ALLOW       Anywhere         # 3306 포트가 ALLOW인지 확인 (IPv4)           
3306 (v6)                  ALLOW       Anywhere (v6)    # 3306 포트가 ALLOW인지 확인 (IPv6)            
```
* [source] - bind-address 설정 확인
    * `0.0.0.0` 또는 replica IP로 설정
    * 127.0.0.1 이면 내부에서만 접속 가능
```bash
grep -r "^bind-address" /etc/mysql
```
```bash
/etc/mysql/mysql.conf.d/mysqld.cnf:bind-address = 0.0.0.0 # 모든 IP 허용(외부 접속 가능)
```
```bash
# bind-address를 건드리는 부분이 여러 개가 나온다면 설정파일을 적용하는 순서도 확인하기!
mysqld --help | grep -A 1 "Default options"
mysqld --verbose --help | grep -A 5 "Default options"
```
* [replica] - 네트워크 연결 확인
```bash
ping 192.168.111.129        # source IP
telnet 192.168.111.129 3306 # source IP, MySQL port
```
<br>

## 3. secure connection 없이 caching_sha2_password 인증 시도한 경우
```sql
Source_Host             : 192.168.111.129
Replica_IO_Running      : Connecting -- source에 연결을 계속 재시도 중
Replica_SQL_Running     : Yes -- 기존 relay log가 있다면 SQL thread는 계속 동작 가능
Seconds_Behind_Source   : 0
Last_IO_Error           : Error connecting to source 'repl@192.168.111.129:3306'. This was attempt 9/86400, with a delay of 60 seconds between attempts. Message: Authentication plugin 'caching_sha2_password' reported error: Authentication requires secure connection.
129:3306' (110)
Last_SQL_Error          : 
```
### 현상
* 네트워크 연결을 시도했지만 인증 단계에서 secure connection 요구 조건 때문에 실패
* 네트워크가 끊긴 것이 아니라, 인증 단계에서 실패
### 원인
* caching_sha2_password는 MySQL 8.x에서 기본 인증 플러그인
* 보안 연결(TLS)이 없으면 복제 연결에서 RSA 공개키 방식을 추가로 설정해야 함
* replica가 source에 보안 연결 없이 접속을 시도해서 문제 발생
### 해결
* TLS/SSL 기반 보안 연결 사용
* 보안 연결을 사용하지 않을 경우, replica에서 source 공개키를 요청하도록 설정
```sql
change replication source to
    source_user='binlog_reader',
    source_host='192.168.111.129',
    source_port=3306,
    source_password='aaaa',
    source_auto_position=1,
    get_source_public_key=1; -- source 공개키를 요청
```
* 설정 후 replication 재시작 필요
<br>

## 4. replica가 필요로 하는 binlog가 없는 경우
```sql
Source_Host             : 192.168.111.129
Replica_IO_Running      : Connecting -- source에 재접속을 시도하지만, 실제로는 필요한 binlog 이력이 없어서 진행 불가
Replica_SQL_Running     : Yes -- 기존 relay log가 있다면 SQL thread는 남아 있을 수 있음
Seconds_Behind_Source   : 0
Last_IO_Error           : Got fatal error 1236 from source when reading data from binary log: 'Cannot replicate because the source purged required binary logs. Replicate the missing transactions from elsewhere, or provision a new replica from backup. Consider increasing the source's binary log expiration period. The GTID set sent by the replica is '82141e86-396c-11f1-8939-000c29ecd7e8:1-13', and the missing transactions are 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:14-33''
Last_SQL_Error          : 
```
### 현상
* source의 binlog/GTID 이력이 삭제되어 replica가 source를 따라갈 수 없는 현상
### 원인
* replica가 source에 접속해 GTID 기반으로 이어서 복제를 시도하지만 필요한 트랜잭션이 들어 있던 source의 binary log가 이미 purge됨
### 해결
* 누락된 트랜잭션을 다른 replica나 보관된 binlog에서 가져와 적용
* 또는 source의 최신 백업으로 replica를 재구성
* 재발 방지를 위해 source의 binary log 보존 기간 늘리기(`binlog_expire_logs_seconds`)
<br>

### source의 최신 백업으로 replica를 재구성하는 방법
* [replica] - replication 중단 및 초기화
```sql
stop replica; -- replication 중단
reset replica all; -- replication 메타데이터 초기화

-- replica에 기존 데이터가 남아 있다면 source dump와 충돌하지 않도록 정리 필요
-- database, table, 사용자 환경에 따라 datadir 재구성이 필요할 수도 있음
```
* [source] - dump 생성
    * InnoDB 기준 일관된 snapshot + GTID 정보를 포함한 dump 생성
```bash
sudo mysqldump -u root -p --all-databases --triggers --routines --events --single-transaction --set-gtid-purged=ON > full.sql
```
* [source] - dump 파일을 replica로 복사
```bash
scp full.sql dbadmin@192.168.111.130:/home/dbadmin/
# dbadmin : replica 서버의 사용자 계정
# 192.168.111.130 : replica 서버 IP
# /home/dbadmin/ : replica 서버에서 파일을 저장할 경로
```
* [replica] - dump 파일 import
```bash
sudo mysql -uroot -p < /home/dbadmin/full.sql
```
* [replica] - replication 연결 및 시작
```sql
change replication source to
    source_user='binlog_reader',    -- source에 존재하는 replication 인증 계정
    source_host='192.168.111.129',  -- 접속할 source IP
    source_port=3306,               -- MySQL 포트
    source_password='aaaa',         -- binlog_reader 계정 비밀번호
    source_auto_position=1,         -- 실제 replication 연결을 GTID 방식으로 수행하도록 설정
    get_source_public_key=1;        -- source 공개키 요청

start replica;
```
* [replica] - replication 상태 확인
```sql
show replica status\G
```