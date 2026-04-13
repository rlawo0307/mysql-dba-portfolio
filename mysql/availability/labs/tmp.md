# 목차
* replication 구축하기
    * 같은 서버에서 멀티 인스턴스 구축
    * source 설정
        * 설정 파일 작성 및 반영
        * replica 계정 생성 및 권한 부여
    * replica 설정
        * 설정 파일 작성 및 반영
        * replication 수행 시 source에 접속할 계정 설정
        * replication 연결 확인
    * 초기 데이터 동기화
    * replication 연결
    * replication 중단 및 재시작
* 참고 자료
<br><br>

# Replication 구축하기
* source의 변경 사항이 replica에 **비동기**로 복제되는지 확인해보자
* binlog 기반 replication이 실제로 어떤 흐름으로 동작하는지 확인해보자
* GTID 기반으로 replication을 구성하여 이후 failover 실습의 기반을 마련해보자
## 0. 같은 서버에서 멀티 인스턴스 구축
#### 1) 인스턴스별 별도의 설정 파일 사용하기
* source 용 : `/etc/mysql/source.cnf`
* replica 용 : `/etc/mysql/replica.cnf`
```cnf
# /etc/mysql/source.cnf

[mysqld]
port=3306                                   # MySQL 접속 포트 분리
datadir=/var/lib/mysql-source               # datadir 분리
pid-file=/var/run/mysqld/mysqld-source.pid  # pid-file 분리
socket=/var/run/mysqld/mysqld-source.sock   # socket 분리
log-error=/var/log/mysql/source-error.log

server_id=1
log_bin=mysql-bin-source
```
```cnf
# /etc/mysql/replica.cnf

[mysqld]
port=3307
datadir=/var/lib/mysql-replica
pid-file=/var/run/mysqld/mysqld-replica.pid
socket=/var/run/mysqld/mysqld-replica.sock
log-error=/var/log/mysql/replica-error.log

server_id=2
log_bin=mysql-bin-replica
```
#### 2) datadir 및 로그 디렉토리 생성
```bash
sudo mkdir -p /var/lib/mysql-source
sudo mkdir -p /var/lib/mysql-replica
sudo mkdir -p /var/log/mysql
sudo mkdir -p /var/run/mysqld

sudo chown -R mysql:mysql /var/lib/mysql-source
sudo chown -R mysql:mysql /var/lib/mysql-replica
sudo chown -R mysql:mysql /var/log/mysql
sudo chown -R mysql:mysql /var/run/mysqld
```
#### 3) 초기 DB 초기화
* 각각 datadir에 MySQL 시스템 테이블을 생성
```bash
sudo mysqld --defaults-file=/etc/mysql/source.cnf --initialize --user=mysql
sudo mysqld --defaults-file=/etc/mysql/replica.cnf --initialize --user=mysql
```
#### 4) 인스턴스 실행
* 기존 systemctl 말고 직접 실행
```bash
sudo mysqld --defaults-file=/etc/mysql/source.cnf --user=mysql &
sudo mysqld --defaults-file=/etc/mysql/replica.cnf --user=mysql &
```
#### 5) MySQL 접속
```bash
# source
mysql -S /var/run/mysqld/mysqld-source.sock -u root -p

# replica
mysql -S /var/run/mysqld/mysqld-replica.sock -u root -p
```
#### 6) 접속 확인
```sql
select @@port, @@socket, @@server_id;
```
## 1. source 설정
* source는 binlog가 활성화 되어 있어야 함
    * MySQL 8.0에서는 binary logging이 기본적으로 활성화되어 있음
    * 명시적으로 `log_bin` 이름을 붙여주는 것을 권장 
* replication 구분을 위해 source, replica 모두 고유한 `server_id`가 필요
* source 계정은 생성할 필요 없음
    * replica → source 단방향 연결 구조
    * replica가 source에 접속해서 binlog를 읽어오는 pull 방식
    * 접속하는 쪽(replica) 계정만 필요
#### 1) 설정 파일 작성 (`/etc/mysql/mysql.conf.d/mysqld.cnf`)
#### 1) 설정 파일 작성
```text
# /etc/mysql/source.cnf

[mysqld]
port=3306
datadir=/var/lib/mysql-source
pid-file=/var/run/mysqld/mysqld-source.pid
socket=/var/run/mysqld/mysqld-source.sock

server_id=1                     # server id
log_bin=mysql-bin-source        # binary logging 활성화
binlog_format=ROW               # RBR 방식 적용
gtid_mode=ON                    # GTID 모드 활성화
enforce_gtid_consistency=ON     # GTID와 호환되지 않는 SQL은 금지
log_replica_updates=ON          # replica가 적용한 변경도 binlog에 기록하도록 설정
```
#### 2) 반영
```bash
sudo systemctl restart mysql
```
#### 3) 확인
```sql
show variables like 'server_id';
show variables like 'log_bin';
show variables like 'binlog_format';
show variables like 'gtid_mode';
show variables like 'enforce_gtid_consistency';
show variables like 'log_replica_updates';
```
#### 4) replica 계정 생성 및 권한 부여
```sql
create user 'repl'@'localhost' identified by 'aaaa';
grant replication slave on *.* to 'repl'@'localhost'; -- replica가 source에 접속하여 binlog를 읽기 위해 사용하는 replication 계정
grant replication_slave_admin on *.* to 'repl'@'localhost';
grant replication client on *.* to 'repl'@'localhost';


flush privileges;

show grants for 'repl'@'localhost';
```
```sql
+--------------------------------------------------------------------------+
| Grants for repl@localhost                                                |
+--------------------------------------------------------------------------+
| GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO `repl`@`localhost` |
| GRANT REPLICATION_SLAVE_ADMIN ON *.* TO `repl`@`localhost`               |
+--------------------------------------------------------------------------+
2 rows in set (0.00 sec)
```
## 2. replica 설정
* replication 구분을 위해 source, replica 모두 고유한 `server_id`가 필요
#### 1) 설정 파일 작성 (`/etc/mysql/mysql.conf.d/mysqld.cnf`)
#### 1) 설정 파일 작성
```text
# /etc/mysql/replica.cnf

[mysqld]
port=3307
datadir=/var/lib/mysql-replica
pid-file=/var/run/mysqld/mysqld-replica.pid
socket=/var/run/mysqld/mysqld-replica.sock

server_id=2                     # server id
log_bin=mysql-bin-source        # binary logging 활성화
binlog_format=ROW               # RBR 방식 적용
gtid_mode=ON                    # GTID 모드 활성화
enforce_gtid_consistency=ON     # GTID와 호환되지 않는 SQL은 금지
log_replica_updates=ON          # replica가 적용한 변경도 binlog에 기록하도록 설정
read_only=ON
# super_read_only=ON

```
#### 2) 반영
```bash
sudo systemctl restart mysql
```
#### 3) replication 수행 시 source에 접속할 계정 설정
* 설정파일(.cnf)에서 설정한 `GTID_MODE=ON`은 GTID 모드를 활성화하겠다는 의미
* `source_auto_position=1`은 실제로 GTID 기반 replication을 수행하겠다는 의미
    * `GITD_MODE=ON` + `source_auto_position=0`
        * GTID 기능은 켜져 있음
        * replication 연결 자체는 GTID 방식으로 안붙을 수도 있음
    * `GITD_MODE=OFF` + `source_auto_position=1`
        * 아예 GTID 기반 replication을 정상적으로 사용할 수 없음
```sql
stop replica;

change replication source to
    source_user='repl',         -- source에 접속할 replica 계정
    source_host='localhost',    -- 접속할 source host
    source_port=3306,         -- MySQL 포트
    source_password='aaaa',     -- replica 계정 비밀번호
    source_auto_position=1;     -- 실제 replication 연결을 GTID 방식으로 수행하도록 설정

start replica;
```
#### 4) replication 연결 확인
```sql
show replica status;
```
```sql
Source_Host             : localhost -- replica가 접속하려는 source 주소
Replica_IO_Running      : NO -- source에서 binlog를 가져오는 thread 상태
Replica_SQL_Running     : Yes -- relay log를 실행하는 thread 상태
Seconds_Behind_Source   : 0 -- replication 지연 시간
Last_IO_Error           : source and replica have equal MySQL server ids -- replication 실패 원인
Last_SQL_Error          : (빈 값) -- SQL 실패 원인
Retrieved_Gtid_Set      : -- source에서 가져온 트랜잭션
Executed_Gtid_Set       : aaa902cd-ecb6-11f0-bda3-000c2921ba33:1-13 -- 실제 적용된 트랜잭션
```
## 3. 초기 데이터 동기화
* 이미 source에 데이터가 있다면 replica를 먼저 같은 상태로 맞춰야 함
#### 테스트용 데이터 준비 (source)
```sql
create database db1;
use db1;

create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);

select * from t1;
```
#### dump 생성 (source)
```bash
mysqldump -uroot -p --single-transaction --set-gtid-purged=ON db1 > db1.sql
```
#### dump 파일을 replica로 복사 (source)
```bash
scp db1.sql repl@localhost:/tmp/db1.sql
```
#### dump 파일 import (replica)
```bash
muysql -uroot -p < /tmp/db1.sql
```
#### 데이터 확인 (replica)
```sql
use db1;
select * from t1;
```
## 4. replication 연결
#### 추가 데이터 입력 (source)
```sql
insert into t1 values (4, 400);
update t1 set c2 = 999 where c1 = 2;
delete from t1 where c1 = 3;

select * from t1;
```
#### 변경된 데이터 확인 (replica)
```sql
select * from t1;
```
## 5. replication 중단 및 재시작
#### replication 중단 (replica)
```sql
stop replica;
```
#### 데이터 변경 (source)
```sql
insert into t1 values(5, 500);

select * from t1;
```
#### 데이터 확인 (replia)
```sql
select * from t1;
```
#### replication 재시작 후 변경된 데이터 확인(replica)
```sql
start replica;

select * from t1;
```