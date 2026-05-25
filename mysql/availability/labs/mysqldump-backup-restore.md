# 목차
* mysqldump를 이용한 logical backup 및 restore 확인
* 참고 자료
    * [backup & recovery](../backup-and-recovery.md)
<br><br>

# mysqldump를 이용한 logical backup 및 restore 확인
* mysqldump를 이용해 logical backup을 생성해보자
* 백업 파일이 실제로 restore 가능한지 확인해보자
* restore 후 database, table, row data가 정상적으로 복구되는지 확인해보자
## 실습 준비
```sql
create database db_test;
use db_test;

create table t1(c1 int primary key, c2 int, c3 varchar(100));
insert into t1 values (1, 100, 'aaa');
insert into t1 values (2, 200, 'bbb');
insert into t1 values (3, 300, 'ccc');

select * from t1;
```
```sql
+----+------+------+
| c1 | c2   | c3   |
+----+------+------+
|  1 |  100 | aaa  |
|  2 |  200 | bbb  |
|  3 |  300 | ccc  |
+----+------+------+
3 rows in set (0.00 sec)
```
## mysqldump로 logical backup 생성
```bash
mysqldump -uuser1 -p --single-transaction --set-gtid-purged=OFF db_test > db_test.sql
# -uuser1               : user1 계정으로 MySQL 접속
# -p                    : 비밀번호 입력 요청   
# --single-transaction  : 하나의 트랜잭션 snapshot 기준으로 일관된 백업 
# --set-gtid-purged=OFF : dump 파일에 GTID 정보 제외
# db_test               : 백업 대상 데이터베이스
# db_test.sql           : dump 파일명
```
## 백업 파일 확인
### 파일 정보 확인
```bash
ls -lh db_test.sql                  
```
```bash
-rw-rw-r-- 1 dbadmin dbadmin 1.9K May 25 02:42 db_test.sql # 파일 크기가 0B라면 생성 실패
```
### 파일 헤더 확인
```bash
head -n 5 db_test.sql
```
```bash
-- MySQL dump 10.13  Distrib 8.0.45, for Linux (x86_64) # mysqldump가 만든 파일인지 
-- Host: localhost    Database: db_test                 # 백업 대상 데이터베이스가 맞는지
-- Server version	8.0.45-0ubuntu0.24.04.1             
```
### table 구조 및 데이터 확인
```bash
grep -n "CREATE TABLE" db_test.sql  # 파일에서 테이블 생성 DDL 검색
grep -n "INSERT INTO" db_test.sql   # 파일에서 데이터 삽입 DML 검색
```
```bash
25:CREATE TABLE `t1` # 파일 25번째 줄에 발견
39:INSERT INTO `t1` VALUES (1,100,'aaa'),(2,200,'bbb'),(3,300,'ccc'); # 파일 39번째 줄에 발견
```
## 원본 데이터 삭제 후 restore 수행
* 기존 데이터가 없는 상태에서도 dump 파일만으로 정상 복구가 가능한지 확인하기 위해 원본 데이터베이스를 삭제
* restore 대상 데이터베이스가 필요하므로 동일한 이름의 비어있는 데이터베이스 생성
```sql
drop database db_test;
create database db_test;
```
```bash
mysql -uuser1 -p db_test < db_test.sql
```
## restore 결과 확인
```sql
use db_test; -- 데이터베이스 전환

show tables; -- table 확인
select * from t1; -- 데이터 확인
```
```sql
+-------------------+
| Tables_in_db_test |
+-------------------+
| t1                | -- 데이터베이스(db_test)에 t1이 존재
+-------------------+
1 row in set (0.00 sec)
```
```sql
+----+------+------+
| c1 | c2   | c3   |
+----+------+------+
|  1 |  100 | aaa  | -- restore 후 기존 row data와 동일하게 복구됨
|  2 |  200 | bbb  |
|  3 |  300 | ccc  |
+----+------+------+
3 rows in set (0.00 sec)
```