# PRIMARY KEY
* 테이블 내 각 행을 유일하게 식별하는 컬럼 또는 컬럼 조합
* 하나의 테이블에는 반드시 하나의 primary key가 존재
* 여러 컬럼으로 구성된 복합 기본키는 가능 → `composite primary key`
* 중복 불가
* null 불가 → 명시하지 않아도 자동으로 `not null`
* **자동으로 unique 인덱스 생성**
### 단일, 복합 primary key 설정
```sql
create table t1 (c1 int primary key, c2 int);
create table t2 (c1 int, c2 int, primary key (c1, c2));
```
### 중복된 pk 값 입력 시 에러 확인
```sql
insert into t1 values (1, 1); -- success
insert into t1 values (1, 2); -- ERROR 1062 (23000): Duplicate entry '1' for key 't1.PRIMARY'
```
### not null 확인
```sql
insert into t1 values (null, 1);
-- ERROR 1048 (23000): Column 'c1' cannot be null
```
### 인덱스 확인
```sql
select table_name, non_unique, index_name, column_name
from information_schema.statistics
where table_name = 't1';
-- show index from t1;
```
```sql
+------------+------------+------------+-------------+
| TABLE_NAME | NON_UNIQUE | INDEX_NAME | COLUMN_NAME |
+------------+------------+------------+-------------+
| t1         |          0 | PRIMARY    | c1          |
+------------+------------+------------+-------------+
1 row in set (0.00 sec)
```
<br>

# DEFAULT
* insert 시 컬럼 값이 명시되지 않았을 경우 자동으로 지정되는 기본값
* **새로 insert 되는 행에만 적용, 기존 행 값은 절대 변하지 않음**
* null을 삽입할 경우 default가 적용되지 않음
* `text`, `blob` 계열은 default 값으로 설정 불가능 (MySQL 8.0.12 이하 기준)
* 데이터 무결성이 아닌 입력 편의를 위한 기능
    * 데이터 무결성 : 잘못된 데이터가 저장되지 않도록 강제하는 것
    * `default`는 값이 생략된 경우에만 적용되며
    * 잘못된 값을 차단하지도, DB 규칙을 강제하지도 않음
```sql
create table t1 (c1 int default 0, c2 int);
create table t2 (c1 text default 'aaa'); -- ERROR 1101 (42000): BLOB, TEXT, GEOMETRY or JSON column 'c1' can't have a default value
```
```sql
insert into t1 (c2) values (1);
insert into t1 values (null, 1);
select * from t1;
+------+------+
| c1   | c2   |
+------+------+
|    0 |    1 | -- c1에 default 적용됨
| NULL |    1 | -- null값을 명시했으므로 default 적용X
+------+------+
2 rows in set (0.00 sec)
```
<br>

# NOT NULL
* 컬럼에 null 값을 저장하지 못하도록 강제
* 반드시 값이 존재해야 하는 컬럼에 사용
* insert 시 컬럼 생략 불가능
    * **`default`가 있다면 생략 가능**
* update 시에도 not null 컬럼은 null 값을 가질 수 없음
* 테이블 생성 이후 `not null` 제약 조건을 추가할 수 있음
    * 단, 이미 null인 데이터가 들어있다면 불가능
* **숫자 0이나 빈 문자열('')로 `not null`을 대신하지 않도록 주의**
### not null 조건으로 테이블 생성
```sql
create table t1 (c1 char not null);

create table t2 (c1 int);
alter table t2 modify c1 int not null;

create table t3 (c1 int not null default 0, c2 int);
```
### insert 시 에러 확인
```sql
insert into t1 values (''); -- success
```
```sql
insert into t1 values (null); -- ERROR 1048 (23000): Column 'c1' cannot be null
```
```sql
insert into t2 values (null); -- ERROR 1048 (23000): Column 'c1' cannot be null
```
```sql
insert into t3 (c2) values (1); -- success
select * from t3;
+------+------+
| c1   | c2   |
+------+------+
|    0 |    1 | -- c1에 지정한 값이 없으니 default로 0
+------+------+
1 row in set (0.00 sec)
```
### update 시 에러 확인
```sql
update t3 set c1 = null where c1 = 0; -- ERROR 1048 (23000): Column 'c1' cannot be null
```
### not null 제약 삭제
```sql
alter table t1 modify column c1 char;
```
<br>

# UNIQUE
* 단일/복합 컬럼에 중복된 값이 들어오는 것을 방지
* 하나의 테이블에 여러 개의 unique 제약을 둘 수 있음
* 테이블 생성 이후 unique 제약 조건을 추가할 수 있음
    * 단, 이미 중복된 데이터가 들어있다면 불가능
* `null` 값은 여러개 존재해도 중복으로 보지 않음
    * null은 값으로 취급하지 않음
    * 즉, 비교대상이 아예 없음
* 내부적으로는 `unique 인덱스`로 구현됨
    * unique 제약 조건은 unique 인덱스를 생성하는 것과 동일
### unique 제약 조건으로 테이블 생성
```sql
create table t1 (c1 int unique, c2 int);
create table t2 (c1 int, c2 int, unique(c1, c2));
create table t3 (c1 int, c2 int, unique(c1), unique(c2));
```
### 여러개의 null 값 삽입
#### 단일 컬럼에 unique 제약이 있을 경우
```sql
insert into t1 values (null, 1);
insert into t1 values (null, 2);
select * from t1;
+------+------+
| c1   | c2   |
+------+------+
| NULL |    1 |
| NULL |    2 |
+------+------+
2 rows in set (0.01 sec)
```
#### 복합 컬럼에 unique 제약이 있을 경우
```sql
insert into t2 values (null, null);
insert into t2 values (null, null); -- success

insert into t2 values (null, 0);
insert into t2 values (null, 0); -- success
```
* 같은 데이터를 중복해서 넣었지만 null을 중복으로 판단하지 않으므로 성공
* 논리적으로는 `(c1, c2)는 유일해야 한다`를 의도했을지라도, 실제로는 둘 중 하나가 null이면 중복 값을 무한히 허용
* **`(?, null)` 또는 `(null, ?)` 조합이 무한이 삽입될 수 있으므로 주의!**
* 복합 unique를 사용할 경우, 유일성을 보장해야 하는 컬럼은 `not null`로 선언하는 것이 안전
### 인덱스 확인
```sql
select table_name, non_unique, index_name, column_name
from information_schema.statistics
where table_name = 't1';
```
```sql
+------------+------------+------------+-------------+
| TABLE_NAME | NON_UNIQUE | INDEX_NAME | COLUMN_NAME |
+------------+------------+------------+-------------+
| t1         |          0 | c1         | c1          |
+------------+------------+------------+-------------+
1 row in set (0.01 sec)
```
* `NON_UNIQUE = 0` : unique index
* 인덱스 이름을 따로 지정하지 않았으므로 컬럼명과 동일한 인덱스 생성
### 컬럼명과 다른 unique 인덱스 생성
```sql
alter table t1 add constraint idx1 unique(c1);
```
```sql
select table_name, non_unique, index_name, column_name
from information_schema.statistics
where table_name = 't1';
```
```sql
+------------+------------+------------+-------------+
| TABLE_NAME | NON_UNIQUE | INDEX_NAME | COLUMN_NAME |
+------------+------------+------------+-------------+
| t1         |          0 | c1         | c1          |
| t1         |          0 | idx1       | c1          |
+------------+------------+------------+-------------+
2 rows in set (0.00 sec)
```
* MySQL은 같은 컬럼에 이름이 다른 인덱스 생성 가능
    * 그럼 실제 index scan 시 어떤 인덱스를 사용할까?
        * 그건 옵티마이저 판단
        * `EXPLAIN` 으로 확인 가능
        * 쓰기 작업 시 모든 인덱스가 갱신되므로 성능 저하를 유발할 수 있음
### unique 제약 삭제
* Oracle과 달리 MySQL에서는 `unique` 제약도 인덱스로 제거
    * Oracle은 `drop constraint`문 사용
```sql
alter table t1 drop index idx1;
```
<br>

# CHECK
* 단일/복합 컬럼 값이 지정된 조건을 만족하는 경우에만 `insert`/`update`를 허용
* delete / select는 무관
* **check 조건으로 서브쿼리나 다른 테이블 참조 불가**
    * 같은 입력 값에 대해 항상 같은 결과를 반환해야 함
    * 현재 행의 값만으로 항상 동일한 결과가 나와야함
    * 외부 데이터나 실행 시점에 따라 결과가 달라지면 안됨
        * now() 같은 일부 함수는 사용 불가능
* 테이블 생성 후 check 제약 추가 가능
* 가능한 단순하고 명확하게 조건을 제시할 것
    * 복잡한 검증은 애플리케이션 또는 트리거에서 처리
* **에러 메시지를 위해 이름 지정 권장**
* MySQL 8.0.15 이하에서는 문법만 허용되고 실제로 강제되지 않음
### check 제약이 있는 테이블 생성
```sql
create table t1 (c1 int, check(c1 >= 0));
create table t2 (c1 int, constraint chk1 check (c1 >= 0)); -- check 제약 이름 지정
create table t3 (c1 int check (c1 between 0 and 10));
create table t4 (c1 int, c2 int, check (c1 <= c2));
```
### check 제약에 위배되는 insert
```sql
insert into t1 values (1); -- success
insert into t1 values (-1); -- ERROR 3819 (HY000): Check constraint 't1_chk_1' is violated.
insert into t1 values (null);
```
* check 제약 이름을 명시하지 않았으므로 자동으로 이름 생성 (`t1_chk_1`)
* check는 `false`만 거부
* null의 연산 결과는 `unknown`이므로 허용
### check 제약 추가
```sql
alter table t1 add constraint chk1 check (c1 < 10);
insert into t1 values (15); -- ERROR 3819 (HY000): Check constraint 'chk1' is violated.
```
### check 제약 확인
```sql
select t.table_name, c.constraint_name, c.check_clause
from information_schema.check_constraints c, information_schema.table_constraints t
where c.constraint_name = t.constraint_name
and c.constraint_schema = t.constraint_schema
and t.table_name = 't1'
and t.constraint_type = 'CHECK';
-- show create table t1;
```
```sql
+------------+-----------------+--------------+
| TABLE_NAME | CONSTRAINT_NAME | CHECK_CLAUSE |
+------------+-----------------+--------------+
| t1         | t1_chk_1        | (`c1` >= 0)  |
| t1         | chk1            | (`c1` < 10)  |
+------------+-----------------+--------------+
2 rows in set (0.00 sec)
```
### check 제약에 위배되는 update
```sql
update t1 set c1 = -1; -- ERROR 3819 (HY000): Check constraint 't1_chk_1' is violated
update t1 set c1 = 15; -- ERROR 3819 (HY000): Check constraint 'chk1' is violated.
```
### check 제약 삭제
```sql
alter table t1 drop check chk1;
```
<br>

# FOREIGN KEY
* 다른 테이블의 기본키를 참조하여 데이터 무결정 유지
