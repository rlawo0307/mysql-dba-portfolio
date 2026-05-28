# 목차
* MySQL to Oracle Migration
    * [source DB] - table 생성 및 데이터 입력
    * [source DB] - CSV export
    * [target DB] - table 생성
    * [source DB] - CSV 파일을 target DB로 복사
    * [target DB] - CSV 파일 확인
    * [target DB] - SQL*Loader 실행
    * 검증
* migration 실패 원인 분석 방법
* 참고 자료
    * [migration이란](../migration.md)
<br><br>

# MySQL to Oracle Migration
## 실습 환경
* source DB
    * Ubuntu Server 24.04 LTS
    * MySQL 8.0.44
    * vmware machine 1
    * ip : `192.168.111.129`
* target DB
    * Ubuntu Server 24.04 LTS
    * Oracle Database 21c XE
    * docker container 기반 실행
    * vmware machine 2
    * ip : `192.168.111.130`
* CSV export / import 기반 logical migration
* scp를 이용해 source → target 으로 CSV 파일 전달
## [source DB] - table 생성 및 데이터 입력
```sql
create table mysql_t1(c1 int primary key auto_increment,
                c2 int,
                c3 varchar(100) not null,
                c4 datetime
);
insert into mysql_t1(c2, c3, c4) values (10, 'aaa', now()),
                                  (20, 'bbb', now()),
                                  (null, 'ccc', now());
```
## [source DB] - CSV export
### CSV export / import 허용 경로 확인
```sql
show variables like 'secure_file_priv';
```
```sql
+------------------+-----------------------+
| Variable_name    | Value                 |
+------------------+-----------------------+
| secure_file_priv | /var/lib/mysql-files/ |
+------------------+-----------------------+
1 row in set (0.01 sec)
```
### CSV export
```sql
select *
into outfile '/var/lib/mysql-files/mysql_t1.csv'
fields terminated by ','
optionally enclosed by '"'
lines terminated by '\n'
from mysql_t1;
```
### CSV 파일 생성 확인
```bash
ls -l /var/lib/mysql-files/mysql_t1.csv
cat /var/lib/mysql-files/mysql_t1.csv
```
```bash
-rw-r----- 1 mysql mysql 99 May 28 08:50 /var/lib/mysql-files/mysql_t1.csv
```
```bash
1,10,"aaa","2026-05-28 08:48:09"
2,20,"bbb","2026-05-28 08:48:09"
3,\N,"ccc","2026-05-28 08:48:09"
```
## [target DB] - table 생성
* oracle `user1` 계정에 table 생성
* source DB의 pk 값을 그대로 유지하기 위해 target table에서는 자동 증가 방식(oracle의 sequence / identity)을 사용하지 않음
    * 운영 전환 이후에 필요 시 자동 증가 정책을 추가로 구성할 수 있음
```sql
create table oracle_t1 (c1 number primary key,
                        c2 number,
                        c3 varchar2(100) not null,
                        c4 timestamp
);
```
|MySQL         |→|Oracle   |
|:------------:|-|:-------:|
|int           | |number   |
|varchar       | |varchar2 |
|datetime      | |timestamp|
|auto_increment| |identity / sequence|
## [source DB] - CSV 파일을 target DB로 복사
```bash
scp /var/lib/mysql-files/mysql_t1.csv dbadmin@192.168.111.130:/tmp/
```
## [target DB] - CSV 파일 확인
### host에서 CSV 파일 확인
```bash
ls -l /tmp/mysql_t1.csv
cat /tmp/mysql_t1.csv
```
```bash
-rw-r----- 1 dbadmin dbadmin 99 May 28 08:55 /tmp/mysql_t1.csv
```
```bash
1,10,"aaa","2026-05-28 08:48:09"
2,20,"bbb","2026-05-28 08:48:09"
3,\N,"ccc","2026-05-28 08:48:09"
```
### Oracle docker container로 복사
```bash
sudo docker cp /tmp/mysql_t1.csv oracle-xe:/tmp/mysql_t1.csv
```
### Oracle docker container 안에서 CSV 파일 확인
```bash
ls -l /tmp/mysql_t1.csv
cat /tmp/mysql_t1.csv
```
```bash
-rw-r----- 1 1000 1000 99 May 28 08:55 /tmp/mysql_t1.csv
```
```bash
1,10,"aaa","2026-05-28 08:48:09"
2,20,"bbb","2026-05-28 08:48:09"
3,\N,"ccc","2026-05-28 08:48:09"
```
## [target DB] - SQL*Loader 실행
### SQL*Loader가 읽을 control file 생성
```bash
cat > /tmp/mysql_t1.ctl << 'EOF'
load data
infile '/tmp/mysql_t1.csv'
into table oracle_t1
fields terminated by ',' optionally enclosed by '"'
trailing nullcols
(
  c1,
  c2 nullif c2 = X'5C4E',
  c3,
  c4 timestamp "YYYY-MM-DD HH24:MI:SS"
)
EOF
```
### SQL*Loader 실행 (데이터 적재)
```bash
sqlldr user1/aaaa@//localhost:1521/xepdb1 \
control=/tmp/mysql_t1.ctl \
log=/tmp/users.log \
bad=/tmp/users.bad
```
## 검증
* 현재 실습에서는 데이터 양이 많지 않아 전체 데이터를 조회하여 검증
* 운영 환경에서는 row count, null count, pk 중복, 문자 깨짐 등 여러 검증을 진행해야 함
### MySQL
```sql
select * from mysql_t1;
```
```sql
+----+------+-----+---------------------+
| c1 | c2   | c3  | c4                  |
+----+------+-----+---------------------+
|  1 |   10 | aaa | 2026-05-28 08:48:09 |
|  2 |   20 | bbb | 2026-05-28 08:48:09 |
|  3 | NULL | ccc | 2026-05-28 08:48:09 |
+----+------+-----+---------------------+
3 rows in set (0.00 sec)
```
### Oracle
```sql
select * from oracle_t1;
```
```sql
	C1	   C2 C3		   C4
---------- ---------- -------------------- ------------------------------
	 1	   10 aaa		   28-MAY-26 08.48.09.000000 AM
	 2	   20 bbb		   28-MAY-26 08.48.09.000000 AM
	 3	      ccc		   28-MAY-26 08.48.09.000000 AM
```
<br>

# migration 실패 원인 분석 방법
* migration 실패 시 SQL*Loader의 log / bad file을 통해 원인 분석 가능
    * `users.bad` : 적재 실패한 row 기록
    * `users.log` : 적재 과정 및 error 기록
* 실패 원인을 파악했다면 control file 수정 후 재시도
## 실패 예시
### users.bad
```bash
3,\N,"ccc","2026-05-28 08:48:09" # 해당 row를 적재하는데 실패함
```
### users.log
```bash
Record 3: Rejected - Error on table ORACLE_T1, column C2.   # CSV 파일의 3번째 row c2 컬럼에서 에러 발생
ORA-01722: invalid number                                   # 숫자 컬럼에 숫자로 변환할 수 없는 값이 들어옴
                                                            # MySQL에서 null 값이 '\N'으로 export 됨
                                                            # SQL*Loader가 이를 number 컬럼 값으로 변환하려다 실패

Table ORACLE_T1:
  2 Rows successfully loaded.           # 2개 row 적재 성공
  1 Row not loaded due to data errors.  # 1개 row 적재 실패
  0 Rows not loaded because all WHEN clauses were failed.
  0 Rows not loaded because all fields were null.

```
## 성공 예시
### users.log
```bash
Table ORACLE_T1:
  3 Rows successfully loaded.           # 3개 row 적재 성공
  0 Rows not loaded due to data errors.
  0 Rows not loaded because all WHEN clauses were failed.
  0 Rows not loaded because all fields were null.
```