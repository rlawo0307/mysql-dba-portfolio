# 권한(Privilege)
* 종류
    * `SELECT`
        * 테이블이나 뷰의 데이터를 조회할 수 있는 권한
        * `explain select ..` 쿼리도 select 권한 필요
        * `explain analyze ..` 쿼리도 select 권한이 필요
        * `show ..` 계열 쿼리는 대부분 select 권한이 있는 객체만 조회 가능
            * 일부 명령은 `show view, show index` 등 추가 권한 필요
    * `INSERT`
        * 테이블에 새로운 행을 삽입할 수 있는 권한
        * 기존 행을 수정하는 것은 불가능
        * `on duplicate key update` 옵션 사용 불가능
            * insert + update 권한 필요
    * `UPDATE`
        * 기존 데이터를 수정할 수 있는 권한
    * `DELETE`
        * 테이블의 행을 삭제할 수 있는 권한
    * `CREATE`
        * 객체(테이블, 뷰, 트리거, 프로시저, 함수 등) 생성 권한
            * 인덱스 생성은 index 권한 필요
        * 생성 가능한 객체는 DB 버전과 설정에 따라 다름
        * 일반적으로 DB 단위로 권한 부여
    * `DROP`
        * 객체 삭제 권한
        * `truncate` 실행 가능
            * 내부적으로 drop + create 으로 동작하지만 MySQL은 drop 권한만 검사
    * `ALTER`
        * 테이블 구조를 변경할 수 있는 권한
            * 컬럼 추가, 삭제, 변경
            * 제약조건 변경
    * `INDEX`
        * 인덱스 생성, 삭제, 변경할 수 있는 권한
    * `EXECUTE`
        * stored procedure / 함수를 실행할 수 있는 권한
        * 실행만 가능하고 수정은 불가
            * create 또는 alter 권한 필요
    * `ALL`
        * 해당 범위의 모든 권한 부여
        * all 과 super 권한은 다름
        * 서버 관리 권한은 포함되지 않음
* 범위
    * 상위 범위에서 부여된 권한은 그 아래 모든 범위에서 자동으로 적용됨
    * `서버 > DB > 테이블 > 컬럼`
* 각 권한마다 허용되는 범위가 다름
    * select → 서버, DB, 테이블, 컬럼
    * insert → 서버, DB, 테이블
    * update → 서버, DB, 테이블, 컬럼
    * delete → 서버, DB, 테이블
    * create → 서버, DB
    * drop → 서버, DB
    * alter → DB, 테이블
    * index → DB, 테이블
    * execute → DB
    * all → 서버, DB, 테이블
<br>

# 권한 부여하기 (GRANT)
* MySQL 권한 부여는 **범위는 좁게, 권한을 최소**로 주는 것을 원칙으로 함
* 권한 부여는 권한을 줄 수 있는 권한을 가진 계정(관리자 계정)에서 가능
    * `GRANT OPTION` 권한을 가진 계정
    * root 계정은 모든 권한을 가지고 있으므로 가능
```sql
grant 권한1, 권한2, ...
on 대상
to 사용자 [identified by 비밀번호]
[with grant option];
```
* `grant 권한1, 권한2, ...`
    * 사용자에게 부여할 권한 목록
    * 쉼표(,)로 여러 권한을 동시에 지정 가능
* `on 대상`
    * 권한이 적용되는 범위 지정
    * 권한의 종류에 따라 허용되는 대상의 범위가 다름
* `to 사용자`
    * 권한을 부여받는 계정
    * MySQL에서 계정은 항상 `user+host` 쌍으로 표현
* `[identified by 비밀번호]`
    * 사용자의 비밀번호를 설정
    * MySQL 5.x 이하 버전에서는 계정이 존재하지 않으면 자동으로 계정 생성
    * MySQL 8.x 이후 버전에서는 계정 생성과 권한 부여 쿼리를 분리하여 실행해야 함
* `[with grant option]`
    * 부여 받은 권한을 다른 사용자에게 다시 부여할 수 있는 권한
    * 부여받은 권한과 동일하거나 그보다 좁은 범위로만 재부여 가능
## 여러가지 권한 부여 방법
### 읽기 전용 계정 (조회용)
* 테이블 조회 및 뷰 조회(정의 포함) 권한
* 그 외 다른 DML 불가능
```sql
grant select, show view on 데이터베이스명.* to 사용자;
```
### table 단위 권한
* db1의 t1 테이블에만 접근 가능
```sql
grant select, insert on 데이터베이스명.테이블명 to 사용자;
```
### 컬럼 단위 권한
* 지정한 컬럼만 조회 가능
```sql
grant select (컬럼1, 컬럼2, ...) on 데이터베이스명.테이블명 to 사용자;
```
### DDL 관리 권한
```sql
grant create, alter, drop on 데이터베이스명.* to 사용자;
```
### 모든 DB 조회 권한
```sql
grant select on *.* to 사용자;
```
### DB 생성, 삭제 권한
```sql
grant create, drop on *.* to 사용자;
```
### PROXY 권한
* 사용자2가 사용자1 처럼 행동 가능
* 일반적인 서비스 운영에서는 거의 사용하지 않는 고급 권한
```sql
grant PROXY on 사용자1 to 사용자2;
```
<br>

# 권한 회수하기 (REVOKE)
* 권한을 회수할 수 있는 계정
    * 해당 권한을 가지고 있고, 그 권한에 대해 `grant option`이 있는 계정
    * root 계정
* __권한을 부여한 계정만 권한을 회수할 수 있는 것은 아님__
* 부여받은 범위와 동일하거나 회수 범위가 더 좁아야 정상적으로 회수됨
* 이미 없는 권한을 회수해도 에러가 발생하지 않고 무시됨
```sql
revoke 권한1, 권한2, ...
on 대상
from 사용자;
```
* `on 대상`
    * 회수할 권한이 적용되었던 범위
    * 권한을 부여받을 때 사용한 범위와 동일하거나 회수 범위가 더 좁아야 정상적으로 회수됨
        * 부여받은 범위 ≥ 회수 범위
<br>

# 권한 확인하기
* 해당 사용자가 가진 권한을 확인하는 방법
* 실제 내부 권한은 여러 grant 결과로 나누어 표시될 수 있음
* 하위 객체에 직접 권한이 없어도 상위 범위에 권한이 있으면 접근 가능
### show 명령으로 확인하기
* MySQL이 권한을 평가한 결과를 보여줌
* 상위 범위 권한이 자동으로 반영된 상태
* 실무에서 가장 많이 사용
```sql
show grants for 사용자;
```
### 시스템 뷰를 통해 확인하기
* 권한이 저장된 메타데이터를 조회
* 권한이 저장된 형태 그대로를 보여줌
* 상위 권한이 하위로 자동 전개되지 않음
* 즉, 직접적으로 부여된 권한의 메타데이터만 볼 수 있음
* 종류
    * 서버 범위 권한 조회 : `information_schema.user_privileges`
    * DB 범위 권한 조회 : `information_schema.schema_privileges`
    * 테이블 범위 권한 조회 : `information_schema.table_privileges`
    * 컬럼 범위 권한 조회 : `information_schema.column_privileges`
```sql
select * from information_schema.schema_privileges where grantee = 사용자;
```
