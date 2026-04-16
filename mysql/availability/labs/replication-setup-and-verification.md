# 목차
* replication 구축하기
    * MySQL 서버 구조
    * source 설정
        * 설정 파일 작성 및 반영
        * replication 인증 계정 생성 및 권한 부여
    * replica 설정
        * 설정 파일 작성 및 반영
        * source 접속 방법 및 replication 기준 설정
        * replication 연결 확인
    * 정상 동작 확인
    * replication 중단 및 재시작 후 서버 동기화 확인
<br><br>

# Replication 구축하기
* source의 변경 사항이 replica에 **비동기**로 복제되는지 확인해보자
* binlog 기반 replication이 실제로 어떤 흐름으로 동작하는지 확인해보자
* GTID 기반으로 replication을 구성하여 이후 failover 실습의 기반을 마련해보자
#### `※ 본 실습에서는 각 서버의 설정을 기본적인 replication 동작 확인을 위한 최소 구성을 기준으로 한다. (failover, chained replication 등 추가적인 구성은 고려하지 않음) ※`
## 0. MySQL 서버 구조
* replication을 구축하기 위해서는 2개의 MySQL 서버가 필요
    * 서버1(`192.168.111.129`) 
        * root : source 서버를 관리하는 계정
        * binlog_reader : replica의 replication 접속을 인증하는 계정
    * 서버2(`192.168.111.130`)
        * root : replica 서버를 관리하는 계정
## 1. [source] - 서버 설정
### 1) 설정 파일 작성 (`/etc/mysql/mysql.conf.d/mysqld.cnf`)
* replication 구분을 위해 source, replica 모두 고유한 `server_id`가 필요
* source는 binlog가 활성화 되어 있어야 함
    * MySQL 8.0에서는 binary logging이 기본적으로 활성화되어 있음
    * 명시적으로 `log_bin` 이름을 붙여주는 것을 권장 
```text
[mysqld]
server_id=1                     # server id
log_bin=mysql-bin               # binary logging 활성화
binlog_format=ROW               # RBR 방식 적용
gtid_mode=ON                    # GTID 모드 활성화
enforce_gtid_consistency=ON     # GTID와 호환되지 않는 SQL은 금지
```
### 2) 반영
```bash
sudo systemctl restart mysql
```
### 3) 확인 (root 계정)
```sql
show variables like 'server_id';                -- 1
show variables like 'log_bin';                  -- ON
show variables like 'binlog_format';            -- ROW
show variables like 'gtid_mode';                -- ON
show variables like 'enforce_gtid_consistency'; -- ON
```
<br>

## 2. [source] - replication 인증 계정 생성 및 권한 부여
* source 서버에는 replica 서버의 replication 접속을 인증하는 전용 계정이 필요함 (`binlog_reader` 계정)
    * replica는 이 계정으로 source에 접속하여 binary log를 읽어옴
    * 해당 계정은 source 서버에 존재하지만, 실제 접속 주체는 replica 서버
        * **계정의 host는 replica 서버의 IP 또는 `%`로 설정해야 외부에서 접속 가능함**
        * localhost로 설정할 경우, 동일 서버에서의 접속만 허용하므로 replica 서버에서 접속할 수 없음
    * source 관리 계정(`root`)을 사용하여 replication 연결을 구성할 수도 있지만, 일반적으로 권장되지 않음
        * root 권한 최소화
        * 작업 역할 분리
        * 계정 노출 시 보안 위험 최소화
* binlog_reader 계정에는 `replication slave` 권한이 부여됨
    * source의 binary log를 읽고 복제할 수 있는 권한
```sql
create user 'binlog_reader'@'192.168.111.130' identified by 'aaaa'; -- 192.168.111.130 : replica 서버 IP
grant replication slave on *.* to 'binlog_reader'@'192.168.111.130';
flush privileges;

show grants for 'binlog_reader'@'192.168.111.130';
```
```sql
+---------------------------------------------------------------------+
| Grants for binlog_reader@192.168.111.130                            |
+---------------------------------------------------------------------+
| GRANT REPLICATION SLAVE ON *.* TO `binlog_reader`@`192.168.111.130` |
+---------------------------------------------------------------------+
1 row in set (0.01 sec)
```
<br>

## 3. [replica] - 서버 설정
### 1) 설정 파일 작성 (`/etc/mysql/mysql.conf.d/mysqld.cnf`)
* replication 구분을 위해 source, replica 모두 고유한 `server_id`가 필요
```text
[mysqld]
server_id=2                     # server id
gtid_mode=ON                    # GTID 모드 활성화
enforce_gtid_consistency=ON     # GTID와 호환되지 않는 SQL은 금지
read_only=ON                    # 읽기 전용 서버로 설정
super_read_only=ON              # 강력하게 읽기만 가능하도록 설정
```
### 2) 반영
```bash
sudo systemctl restart mysql
```
### 3) 확인
```sql
show variables like 'server_id';                -- 2
show variables like 'gtid_mode';                -- ON
show variables like 'enforce_gtid_consistency'; -- ON
show variables like 'read_only';                -- ON
show variables like 'super_read_only';          -- ON
```
<br>

## 4. [replica] - source 접속 방법 및 replication 기준 설정
* `stop replica`
    * replication 중지
    * replica thread가 돌고 있는 상태에서 반드시 replication을 중단 후 설정 변경
* `reset replica all`
    * 기존 replication 메타데이터 초기화
    * 기존 source 연결 정보, relay log 정보 등을 삭제
    * 새로 연결하거나 이전 설정 흔적을 없애고 싶을 때 사용
    * 상황에 따라 사용에 주의 필요
* 최초 설정이라면 `stop replica`, `reset replica all`이 꼭 필요하지 않음
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
* 설정파일(.cnf)의 `GTID_MODE=ON` : GTID 모드를 활성화
* `source_auto_position=1` : 실제 replication 연결을 GTID 방식으로 수행하도록 설정
    * GTID를 기준으로 자동으로 복제 위치를 맞춤 
    * binlog file / position을 직접 지정할 필요 없음
```text
[참고]

* GTID_MODE = ON + source_auto_position = 0
    - GTID 기능은 켜져 있음
    - replication 연결 자체는 GTID 방식으로 안붙을 수도 있음

* GTID_MODE = OFF + source_auto_position = 1
    - 아예 GTID 기반 replication을 정상적으로 사용할 수 없음
```
<br>

## 5. [replica] - replication 연결 확인
```sql
show replica status; -- 원하는 컬럼만 직접 조회할 수 없음
```
또는 
```bash
# \G 출력 후 grep으로 필요한 항목만 확인할 수 있음
sudo mysql -e "show replica status\G" | grep -E "Source_Host|Replica_IO_Running|Replica_SQL_Running|Seconds_Behind_Source|Last_IO_Error|Last_SQL_Error"
```
* replication 연결 성공 시 출력 결과 [→ 실패 시 출력 결과 및 해결 방법](tmp1.md)
```sql
Source_Host             : 192.168.111.129   -- replica가 접속하려는 source 주소
Replica_IO_Running      : Yes               -- I/O thread 상태 (source에서 binlog를 가져오는 thread)
Replica_SQL_Running     : Yes               -- SQL thread 상태 (relay log를 실행하는 thread)
Seconds_Behind_Source   : 0                 -- replication 지연 시간
Last_IO_Error           :                   -- replication 실패 원인
Last_SQL_Error          :                   -- SQL 실패 원인
```
<br>

## 6. replication 정상 동작 확인
* source 서버와 replica 서버에서 동일한 데이터가 조회된다면 replication이 정상 동작 중임을 나타냄
### [source] - 데이터 입력
```sql
use db1;
drop table if exists t1;
create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);

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
### [replica] - 변경된 데이터 확인
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
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
<br>

## 7. replication 중단 및 재시작 후 서버 동기화 확인
### [replica] - replication 중단
```sql
stop replica;
```
### [source] - 데이터 변경
```sql
insert into t1 values(4, 400);
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
|    4 |  400 |
+------+------+
4 rows in set (0.00 sec)
```
### [replica] - 데이터 확인
```sql
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
### [replica] - replication 재시작 후 데이터 확인
```sql
start replica;
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
|    4 |  400 |
+------+------+
4 rows in set (0.01 sec)
```