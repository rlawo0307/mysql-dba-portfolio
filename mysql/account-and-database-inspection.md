# 데이터베이스 목록 조회하기
```sql
show databases; -- 전체 데이터베이스 목록 조회
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)

select DATABASE(); -- 현재 데이터베이스 조회
+------------+
| database() |
+------------+
| sys        |
+------------+
1 row in set (0.00 sec)
```
# 계정 조회하기
* MySQL은 접속할 때 정확히 일치하는 계정이 없으면 명단(MySQL 계정 목록)에서 가장 비슷한 계정을 찾아서 씀
    * user(), current_user()가 같다면, 단일 계정 매칭
    * 다르다면, 대체 계정으로 권한 적용 중
## 접속 시도할 때 사용한 계정 조회
```sql
select user();
+----------------+
| user()         |
+----------------+
| root@localhost |
+----------------+
```
## 실제 권한이 적용된 계정 조회
```sql
select current_user();
+----------------+
| current_user() |
+----------------+
| root@localhost |
+----------------+
```

## MySQL 사용자 계정 목록 조회
* 관리자 권한 필요
* 일반 사용자는 조회할 수 없음
```sql
select user, host
from mysql.user
order by user, host;
```
```sql
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
* debian-sys-maint@localhost
    * Debian/Ubuntu에서 MySQL 서비스 관리용
    * 패키지 업데이트나 서비스 재시작 시 사용
    * 지우면 안됨
    * (로그인은 가능하지만) 일반 사용자가 직접 사용하는 계정은 아님
* mysql.infoschema@localhost
    * information_schema 내부용
    * 로그인 불가
* mysql.session@localhost
    * MySQL 내부 세션용 계정
    * 로그인 불가
* mysql.sys@localhost
    * sys 데이터베이스 뷰의 DEFINER
    * 성능/상태 조회용
    * 로그인 불가
* root@localhost
    * 관리자 계정
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