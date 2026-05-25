# 목차
* PITR(Point-in-Time Recovery) 재현 실습
    * binary log 관련 옵션 설정
    * 정상 작업 중 사고 발생 상황 재현
    * full restore
    * 특정 시점까지 replay
    * 최종 검증
* 참고 자료
    * [여러 log 종류](../../administration/logging.md)
    * [binary log란?](../binlog-and-GTID.md)
    * [mysqldump를 사용한 full restore](mysqldump-backup-restore.md)
<br><br>

# PITR(Point-in-Time Recovery) 재현
* full backup + binlog를 사용해서 특정 시점까지 복구를 진행해보자
## binary log 관련 옵션 설정
* PITR 수행을 위해 backup 이후 변경 내역을 binary log에 기록할 수 있도록 활성화 필요
* PITR에 적합한 row-based 방식으로 기록하도록 설정 필요
### 옵션이 설정된 파일 위치 확인
* 설정 되어 있지 않다면 설정 파일(.cnf)에 추가
```bash
grep -R -E "log_bin|binlog_format" /etc/mysql /etc/my.cnf ~/.my.cnf 2>/dev/null
```
### 옵션 설정 확인 
```sql
show variables like 'log_bin';
show variables like 'binlog_format';
```
```sql
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    | -- binary log 활성화
+---------------+-------+
1 row in set (0.01 sec)
```
```sql
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   | -- row-based 방식으로 기록
+---------------+-------+
1 row in set (0.00 sec)
```
### 현재 binlog 파일 확인
```sql
show master status; -- binary log 상태 확인 명령어
```
```sql
File                : mysql-bin.000034 -- 현재 기록중인 binlog 파일명
Position            : 253 -- 현재 binary log 내부 기록 위치
Binlog_Do_DB        : 
Binlog_Ignore_DB    :
Executed_Gtid_set   : aaa902cd-ecb6-11f0-bda3-000c2921ba33:1-299 -- MySQL 서버 UUID + 현재 서버에서 실행 완료된 GTID 트랜잭션 범위
```
## 정상 작업 중 사고 발생 상황 재현
* [정상 작업] - table 생성 후 데이터 삽입
```sql
create database db_test;
use db_test;

create table t1(c1 int primary key, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);
```
* 여기까지 full backup 생성
```bash
mysqldump -uuser1 -p --single-transaction --set-gtid-purged=OFF db_test > full.sql
```
* [정상 작업] - 이후 계속 데이터 작업 수행
```sql
update t1 set c2 = 999 where c1 = 2;
insert into t1 values (4, 400);
```
* [사고 발생] - 데이터 유실 상황 발생
```sql
delete from t1;
```
## full restore
### DB 초기화
* 데이터베이스 삭제 후 동일한 이름의 비어있는 데이터베이스 생성
```sql
drop database db_test; 
create database db_test;
```
### restore
```bash
mysql -uuser1 -p db_test < full.sql
```
### restore 결과 확인
```sql
use db_test; -- 데이터베이스 전환

show tables; -- table 확인
select * from t1; -- 데이터 확인
```
```sql
+----+------+
| c1 | c2   |
+----+------+
|  1 |  100 |
|  2 |  200 |
|  3 |  300 |
+----+------+
3 rows in set (0.00 sec)
```
* full backup을 생성한 시점으로 복구됨
* update, insert (4, 400)은 아직 없음
## 특정 시점까지 replay
### binlog 확인
```bash
sudo mysqlbinlog /var/lib/mysql/mysql-bin.000034
```
```text
.
.
.
# at 1949
#260525  7:00:07 server id 1  end_log_pos 2020 CRC32 0xef870b20 	Delete_rows: table id 97 flags: STMT_END_F

BINLOG '
d/MTahMBAAAANAAAAJ0HAAAAAGEAAAAAAAEAB2RiX3Rlc3QAAnQxAAIDAwACAQEAdGvegA==
d/MTaiABAAAARwAAAOQHAAAAAGEAAAAAAAEAAgAC/wABAAAAZAAAAAACAAAA5wMAAAADAAAALAEA
AAAEAAAAkAEAACALh+8=
'/*!*/;
### DELETE FROM `db_test`.`t1`
### WHERE
###   @1=1 /* INT meta=0 nullable=0 is_null=0 */
###   @2=100 /* INT meta=0 nullable=1 is_null=0 */
### DELETE FROM `db_test`.`t1`
### WHERE
###   @1=2 /* INT meta=0 nullable=0 is_null=0 */
###   @2=999 /* INT meta=0 nullable=1 is_null=0 */
### DELETE FROM `db_test`.`t1`
### WHERE
###   @1=3 /* INT meta=0 nullable=0 is_null=0 */
###   @2=300 /* INT meta=0 nullable=1 is_null=0 */
### DELETE FROM `db_test`.`t1`
### WHERE
###   @1=4 /* INT meta=0 nullable=0 is_null=0 */
###   @2=400 /* INT meta=0 nullable=1 is_null=0 */
```
* delete 이벤트의 시작 position : `1949`
* delete 이벤트의 종료 position : `2020`
* 삭제된 row
    * @1 = 1 / @2 = 100
    * @1 = 2 / @2 = 999
    * @1 = 3 / @2 = 300
    * @1 = 4 / @2 = 400
### replay 수행
* delete 이벤트 시작 position(`1949`)을 stop-position으로 지정하여 해당 이벤트 이전 변경 내역만 replay
* GTID가 활성화된 환경에서는 binlog replay 시, 기존 GTID가 이미 실행된 트랜잭션으로 인식될 수 있음
    * `--skip-gtids` 옵션으로 GTID 정보를 제외하고 변경 SQL만 replay 하도록 설정
```bash
sudo mysqlbinlog --skip-gtids --stop-position=1949 /var/lib/mysql/mysql-bin.000034 | mysql -uuser1 -p
```
## 최종 검증
```sql
select * from t1;
```
```sql
+----+------+
| c1 | c2   |
+----+------+
|  1 |  100 |
|  2 |  999 |
|  3 |  300 |
|  4 |  400 |
+----+------+
4 rows in set (0.00 sec)
```
* full backup 이후 수행된 정상 작업(update, insert)은 복구됨
* 사고가 발생한 delete는 제외됨