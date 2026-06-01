# Role 생성하기 (CREATE ROLE)
* 권한이 있는 계정에서 생성 가능
* role은 권한을 담는 객체이므로 직접 로그인할 수 없음
```sql
create role 'role명';
```
<br>

# Role 삭제하기 (DROP ROLE)
* role에 부여된 권한 정보도 함께 제거됨
* 사용자에게 부여된 role 연결도 제거됨
```sql
drop role 'role명';
```
<br>

# GRANT / REVOKE를 이용한 Role 관리
```sql
grant 권한 on 대상 to 'role명';         -- role에 권한 부여
revoke 권한 on 대상 from 'role명';      -- role에 부여된 권한 회수
```
```sql
grant 'role명' to '사용자명'@'host';    -- 사용자에게 role 부여
revoke 'role명' from '사용자명'@'host'; -- 사용자로부터 role 회수
```
<br>

# Role 활성화 상태 변경하기 (SET ROLE)
* 부여받은 role은 자동으로 활성화되지 않을 수 있음
* 특정 role만 비활성화하는 명령은 없음
    * 활성화할 role 목록을 다시 지정하여 활성화 상태를 변경해야 함
```sql
set role all;                       -- 모든 role 활성화
set role none;                      -- 모든 role 비활성화
set role default;                   -- default role만 활성화
set role 'role명';                  -- 특정 role 활성화
set role 'role명1', 'role명2', ...; -- 여러 role 활성화
```
<br>

# Default Role 설정하기
* 로그인 시 자동으로 활성화될 role 지정
```sql
set default role 'role명' to '사용자명'@'host';
```
<br>

# 현재 세션에서 활성화된 Role 확인하기
```sql
select current_role();
```

# Role 및 권한 확인하기
```sql
show grants for 'role명';           -- 특정 role 권한 확인
show grants for '사용자명'@'host';  -- 사용자가 가진 role 확인
```