# [권한](../administration/privilege.md) 부여하기 (GRANT)
* 해당 권한을 가지고 있고, 그 권한에 대해 `grant option`이 있는 계정에서 수행 가능
* 일반적으로 root 계정은 모든 권한을 가지고 있으므로 가능
* MySQL 8.0부터는 계정 생성과 권한 부여를 별도로 수행해야 함
```sql
grant 권한1, 권한2, ... on 대상 to 사용자 [with grant option];
```
* `[with grant option]`
    * 부여 받은 권한을 다른 사용자에게 다시 부여할 수 있는 권한
    * 부여받은 권한과 동일하거나 그보다 좁은 범위로만 재부여 가능
### 예시
```sql
grant create user on *.* to 'user1'@'localhost';            -- global 권한
grant select, insert on db1.* to 'user1'@'localhost';       -- database 권한
grant select on db1.t1 to 'user1'@'localhost';              -- table 권한
grant select(c1, c2) on db1.t1 to 'user1'@'localhost';      -- column 권한
grant execute on procedure db1.proc1 to 'user1'@'localhost'; -- routine 권한
```
<br>

# 권한 회수하기 (REVOKE)
* 해당 권한을 가지고 있고, 그 권한에 대해 `grant option`이 있는 계정에서 수행 가능
* 일반적으로 root 계정은 모든 권한을 가지고 있으므로 가능
* **권한을 부여한 계정만 권한을 회수할 수 있는 것은 아님**
* 이미 없는 권한을 회수해도 에러가 발생하지 않고 무시됨
```sql
revoke 권한1, 권한2, ... on 대상 from 사용자;
```
<br>

# 권한 확인하기
* 해당 사용자가 가진 권한을 확인하는 방법
* 권한 정보가 여러 grant 문으로 나누어 표시될 수 있음
* 하위 객체에 직접 권한이 없어도 상위 범위에 권한이 있으면 접근 가능
### show 명령으로 확인하기
* MySQL이 권한을 평가한 결과를 보여줌
* 상위 범위 권한이 자동으로 반영된 상태
```sql
show grants for 사용자;
```
### 시스템 뷰를 통해 확인하기
* 권한이 저장된 메타데이터를 조회
* 권한이 저장된 형태 그대로를 보여줌
* 상위 권한이 하위로 자동 전개되지 않음
* 실제 권한 판단 결과와 다를 수 있음
* 종류
    * global 범위 권한 조회 : `information_schema.user_privileges`
    * database 범위 권한 조회 : `information_schema.schema_privileges`
    * table 범위 권한 조회 : `information_schema.table_privileges`
    * column 범위 권한 조회 : `information_schema.column_privileges`
```sql
select * from information_schema.schema_privileges;
```