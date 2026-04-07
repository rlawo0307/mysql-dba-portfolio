# 목차
* SQL Security 옵션 비교
    * DEFINER
    * INVOKER
* 참고 자료
    * [→ View, definer, sql security 옵션 자세히 보기](../../sql-commands/view.md)
<br><br>

# SQL Security 옵션 비교
* sql security 옵션 별 오브젝트에 접근하기 위해 어떤 권한이 필요한지 확인해보자
    * `sql security definer` : 뷰를 생성한 사용자 권한으로 실행
    * `sql security invoker` : 뷰를 실행한 사용자 권한으로 실행
## 실습 준비(root)
```sql
-- 데이터베이스 생성 및 접속
create database db1;
use db1;

-- 뷰가 참조할 원본 테이블 생성
create table t1(c1 int); 
insert into t1 values (1), (2), (3);

-- 사용자 계정 생성
create user 'user1'@'localhost' identified by 'aaaa';

-- user1에게 부여된 모든 권한, 권한 위임 능력 전부 제거
revoke all privileges, grant option from 'user1'@'localhost';
grant usage on *.* to 'user1'@'localhost'; -- 명시적으로 아무 권한이 없는 정상 계정 상태로 초기화

show grants for 'user1'@'localhost';
```
```sql
+---------------------------------------------------+
| Grants for user1@localhost                        |
+---------------------------------------------------+
| GRANT USAGE ON *.* TO `user1`@`localhost`         |
+---------------------------------------------------+
1 rows in set (0.00 sec)
```
## DEFINER
### root
```sql
create definer='root'@'localhost' sql security definer view v1 as select * from t1;
create definer='root'@'localhost' sql security definer view v2 as select current_user() as "권한 평가 기준", user() as "쿼리 호출자";

-- user1에게 v1, v2 접근 권한 부여
grant select on db1.v1 to 'user1'@'localhost';
grant select on db1.v2 to 'user1'@'localhost';

show grants for 'user1'@'localhost';
```
```sql
+---------------------------------------------------+
| Grants for user1@localhost                        |
+---------------------------------------------------+
| GRANT USAGE ON *.* TO `user1`@`localhost`         |
| GRANT SELECT ON `db1`.`v1` TO `user1`@`localhost` | -- v1 접근 권한 확인
| GRANT SELECT ON `db1`.`v2` TO `user1`@`localhost` | -- v2 접근 권한 확인
+---------------------------------------------------+
```
### user1
```sql
select * from (select count(*) as num_rows from db1.v1) as v1, db1.v2;
```
```sql
+----------+----------------------+------------------+
| num_rows | 권한 평가 기준         | 쿼리 호출자        |
+----------+----------------------+------------------+
|        3 | root@localhost       | user1@localhost  |
+----------+----------------------+------------------+
```
## INVOKER
### root
```sql
create definer='root'@'localhost' sql security invoker view v1 as select * from t1;
create definer='root'@'localhost' sql security invoker view v2 as select current_user() as "권한 평가 기준", user() as "쿼리 호출자";

-- user1에게 v1, v2 접근 권한 부여
grant select on db1.v1 to 'user1'@'localhost';
grant select on db1.v2 to 'user1'@'localhost';

show grants for 'user1'@'localhost';
```
```sql
+---------------------------------------------------+
| Grants for user1@localhost                        |
+---------------------------------------------------+
| GRANT USAGE ON *.* TO `user1`@`localhost`         |
| GRANT SELECT ON `db1`.`v1` TO `user1`@`localhost` |
| GRANT SELECT ON `db1`.`v2` TO `user1`@`localhost` |
+---------------------------------------------------+
3 rows in set (0.00 sec)
```
### user1
```sql
select * from (select count(*) as num_rows from db1.v1) as v1, db1.v2;
```
```sql
ERROR 1356 (HY000): View 'db1.v1' references invalid table(s) or column(s) or function(s) or definer/invoker of view lack rights to use them
-- 뷰 내부 실행 시 invoker(user1) 권한으로 평가
-- user1에게 원본 테이블 권한이 없어서 실행 단계에서 에러 발생
```
## INVOKER - 원본 테이블 권한 부여 후 동작
### root
```sql
grant select on db1.t1 to 'user1'@'localhost';

show grants for 'user1'@'localhost';
```
```sql
+---------------------------------------------------+
| Grants for user1@localhost                        |
+---------------------------------------------------+
| GRANT USAGE ON *.* TO `user1`@`localhost`         |
| GRANT SELECT ON `db1`.`t1` TO `user1`@`localhost` |
| GRANT SELECT ON `db1`.`v1` TO `user1`@`localhost` |
| GRANT SELECT ON `db1`.`v2` TO `user1`@`localhost` |
+---------------------------------------------------+
```
### user1
```sql
select * from (select count(*) as num_rows from db1.v1) as v1, db1.v2;
```
```sql
+----------+----------------------+------------------+
| num_rows | 권한 평가 기준         | 쿼리 호출자        |
+----------+----------------------+------------------+
|        3 | user1@localhost      | user1@localhost  |
+----------+----------------------+------------------+
1 row in set (0.01 sec)
```
## 결과
* sql security definer : definer 권한으로 뷰 실행 → 성공
* sql security invoker : invoker 권한으로 뷰 실행
    * 원본 테이블 권한이 없는 경우 → 에러 발생
    * 원본 테이블 권한이 있는 경우 → 성공
## 결론
실험 결과, sql security 옵션에 따라 뷰 조회 시 필요한 접근 권한이 다른것을 확인할 수 있다.

sql security definer 옵션이 적용된 경우, 뷰는 definer 권한으로 실행되며 호출자는 원본 테이블에 대한 권한이 없이 뷰 접근 권한만 있으면 조회가 가능하다.
반면, sql security invoker의 경우, 뷰는 invoker(호출자) 권한으로 실행되며 호출자는 뷰 접근 권한 뿐만 아니라 뷰가 참조하는 원본 테이블에 대한 권한도 함께 가지고 있어야 뷰 조회가 가능하다.

뷰 실행은 두 단계로 이루어진다. 먼저 호출자가 뷰에 접근할 수 있는 권한이 있는지 확인하고, 이후 sql serucity 설정에 따라 결정된 권한을 기준으로 뷰 내부 쿼리를 실행한다. 결과적으로 뷰는 단순한 쿼리 저장 객체가 아니라, 실행 시점의 권한 컨텍스트를 변경하는 역할을 수행하는 객체이다.