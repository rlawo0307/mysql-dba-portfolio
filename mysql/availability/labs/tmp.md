# 목차
* 수동 failover 실습
    * MySQL 서버 구조
    * Replication 구축하기
    * 장애 발생 전 상태 확인
    * source 장애 발생
    * replica 1을 새 source로 승격 (promote)
    * replica 2가 새 source를 바라보도록 재구성 (reconfigure)
    * 기존 source를 replica로 재편입 (rejoin)
* 참고 자료
    * [source 1대 + replica 1대로 replication 구축 실습](replication-setup-and-verification.md)
    * [Replication 실패 원인별 트러블슈팅 가이드](replication-failure-cases.md)
<br><br>

# 수동 failover 실습
* MySQL replication 환경에서 source 장애 발생 시 failover를 직접 수행해보자
* 단일 replica 승격이 아닌, 전체 replication topology 재구성 과정을 확인하자
## 0. MySQL 서버 구조
* source
    * ip = `192.168.111.129`
    * server_id = `1`
    * 기존 쓰기 서버
    * 장애 복구 후 replica로 재편입
* replica 1
    * `192.168.111.130`
    * server_id = `2`
    * 승격 대상 (실제 운영 환경에서는 replication lag가 가장 적은 replica를 선택)
* replica 2
    * `192.168.111.132`
    * server_id = `3`
    * follower  역할 유지
## 1. Replication 구축하기
<details>
<summary>source 1대 + replica 2대로 구성</summary>

### 1. 설정 파일 작성
* source
```text
[mysqld]
server_id=1                     # server id
log_bin=mysql-bin               # binary logging 활성화
binlog_format=ROW               # RBR 방식 적용
gtid_mode=ON                    # GTID 모드 활성화
enforce_gtid_consistency=ON     # GTID와 호환되지 않는 SQL은 금지
```
* replica 1
```text
[mysqld]
server_id=2                     # server id
gtid_mode=ON                  
enforce_gtid_consistency=ON    
read_only=ON                    # 읽기 전용 서버로 설정
super_read_only=ON              # 강력하게 읽기만 가능하도록 설정
```
* replica 2
```text
[mysqld]
server_id=3                     # server id
gtid_mode=ON    
enforce_gtid_consistency=ON   
read_only=ON    
super_read_only=ON 
```
### 2. [source] - replication 인증 계정 생성 및 권한 부여
```sql
create user 'binlog_reader'@'%' identified by 'aaaa'; -- 외부 접속 모두 허용                                                      
grant replication slave on *.* to 'binlog_reader'@'%';

-- 운영 환경에서는 % 대신 특정 replica 서버 IP로 제한하는 것을 권장
-- create user 'binlog_reader'@'192.168.111.130' identified by 'aaaa'; -- replica 1
-- create user 'binlog_reader'@'192.168.111.132' identified by 'aaaa'; -- replica 2F
-- grant replication slave on *.* to 'binlog_reader'@'192.168.111.130';
-- grant replication slave on *.* to 'binlog_reader'@'192.168.111.132';

flush privileges;
```
### 3. [replica 1,2] - source 접속 방법 및 replication 기준 설정
```sql
stop replica;
reset replica all;

change replication source to
    source_user='binlog_reader',    -- source에 존재하는 replication 인증 계정
    source_host='192.168.111.129',  -- 접속할 source IP
    source_port=3306,               -- MySQL 포트
    source_password='aaaa',         -- binlog_reader 계정 비밀번호
    source_auto_position=1,         -- 실제 replication 연결을 GTID 방식으로 수행하도록 설정
    get_source_public_key=1;        -- source 서버의 공개키를 받아 인증에 사용

start replica;
```
### 4. [replica 1,2] - replication 연결 확인
```sql
show replica status\G
```
* 연결 성공 시 출력 결과
```sql
Source_Host             : 192.168.111.129   -- replica가 접속하려는 source 주소
Replica_IO_Running      : Yes               -- I/O thread 상태 (source에서 binlog를 가져오는 thread)
Replica_SQL_Running     : Yes               -- SQL thread 상태 (relay log를 실행하는 thread)
Seconds_Behind_Source   : 0                 -- replication 지연 시간
Last_IO_Error           :                   -- replication 실패 원인
Last_SQL_Error          :                   -- SQL 실패 원인
```
</details>

## 2. 장애 발생 전 상태 확인
### [source] - 테이블 생성 및 데이터 삽입
```sql
use db1;
drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200);
```
### [replica 1,2] - 데이터 확인
```sql
use db1;
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
+------+------+
2 rows in set (0.00 sec)
```
## 3. source 장애 발생
### [source] - 의도적으로 source 서버 중지
```bash
sudo systemctl stop mysql
```
### [replica 1,2] - replication 연결 상태 확인
```sql
show replica status\G
```
```sql
Source_Host             : 192.168.111.129
Replica_IO_Running      : Connecting
Replica_SQL_Running     : Yes
Seconds_Behind_Source   : NULL
Last_IO_Error           : Error reconnecting to source 'binlog_reader@192.168.111.129:3306'.This was attempt 1/86400, with a delay of 60 seconds between attempts. Message: Can't connect to MySQL server on '192.168.111.129:3306' (111)
Last_SQL_Error          : Replica_SQL_Running_State: Replica has read all relay log;waiting for more updates
```
* replica 설정은 되어 있지만 source 서버의 MySQL 3306 포트에 접속 실패
* source에 연결할 수 없으므로 I/O thread는 실패
* 이미 받아온 relay log를 적용하는 SQL thread는 계속 동작할 수 있음
## 4. replica 1을 새 source로 승격
```sql
stop replica; -- replication 중단
reset replica all; -- replication 정보 초기화

-- 읽기 전용 해제
set global read_only = off;
set global super_read_only = off;

-- 쓰기 테스트
use db1;
insert into t1 values (3, 300);
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
### replication 인증 계정 생성 및 권한 부여
* failover로 인해 source가 변경되는 경우, 새로운 source 서버에 replication 인증 계정(`binlog_reader`) 필요
* 환경에 따라 계정이 이미 존재하는 경우, 생략 가능
```sql
create user 'binlog_reader'@'%' identified by 'aaaa';
grant replication slave on *.* to 'binlog_reader'@'%';
flush privileges;
```
## 5. replica 2가 새 source를 바라보도록 재구성
```sql 
stop replica; 
reset replica all; 

change replication source to
    source_user='binlog_reader',
    source_host='192.168.111.130',  -- 접속할 새 source IP
    source_port=3306,
    source_password='aaaa',
    source_auto_position=1,
    get_source_public_key=1;

start replica;
show replica status\G
```
```sql
Source_Host             : 192.168.111.130
Replica_IO_Running      : Yes
Replica_SQL_Running     : Yes
Seconds_Behind_Source   : 0
Last_IO_Error           : 
Last_SQL_Error          : 
```
* 연결 실패 시, [`2. source 서버 연결 실패`](replication-failure-cases.md) 가이드 확인
### 데이터 확인
```sql
use db1;
select * from t1; -- (1, 100), (2, 200), (3, 300) 이 조회되면 성공
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
## 6. 기존 source를 replica로 재편입
### 서버 재기동
```bash
sudo systemctl start mysql
```
### replica로 재설정
```sql
-- 읽기 전용 설정
set global read_only = on;
set global super_read_only = on;

stop replica;
reset replica all;

change replication source to
    source_user='binlog_reader',
    source_host='192.168.111.130',  -- 접속할 새 source IP
    source_port=3306,
    source_password='aaaa',
    source_auto_position=1,
    get_source_public_key=1;

start replica;
show replica status\G
```
```sql
Source_Host             : 192.168.111.130
Replica_IO_Running      : Yes
Replica_SQL_Running     : Yes
Seconds_Behind_Source   : 0
Last_IO_Error           : 
Last_SQL_Error          : 
```
### 데이터 확인
```sql
use db1;
select * from t1; -- (1, 100), (2, 200), (3, 300) 이 조회되면 성공
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```