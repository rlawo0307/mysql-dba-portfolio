# 데이터베이스 목록 조회하기
### 전체 데이터베이스 목록 조회
```sql
show databases;
```
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)
```
### 현재 데이터베이스 조회
```sql
select DATABASE();
```
```
+------------+
| database() |
+------------+
| sys        |
+------------+
1 row in set (0.00 sec)
```
<br>

# 데이터베이스 생성하기
* 반드시 root 계정에서만 데이터베이스 생성/삭제 가능한 것은 아님
* 권한만 있다면 일반 계정에서도 가능
### 데이터베이스 생성
```sql
create database db1;
```
### 생성한 데이터베이스 확인
```sql
show databases;
```
```
+--------------------+
| Database           |
+--------------------+
| db1                |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```
<br>

# 데이터베이스 삭제하기
### 데이터베이스 삭제
```sql
drop database db1;
```
### 현재 데이터베이스 확인
```sql
show databases;
```
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)
```
<br>

# 데이터베이스 전환하기
```sql
use mysql;

select user, host from user;
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