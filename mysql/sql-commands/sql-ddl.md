# 테이블 생성하기 (CREATE)
```sql
create table 테이블명 (
    컬럼명 자료형 [제약조건] [comment '...'],
    ...
)
[engine=InnoDB]
[default charset=utf8mb4]
[collate=utf8mb4_0900_ai_ci]
[comment='...'];
```
* 테이블명 (table name)
    * 테이터베이스 내에서 유일해야 함
    * 소문자, 언더스코어(_) 사용 권장
* 컬럼명 (column name)
    * 테이블 내에서 유일해야 함
    * 명확하고 의미 있는 이름 사용 권장
* 자료형 (data type) [→ 각 자료형에 대한 자세한 내용 보기](data-types-theory.md)
    * 숫자형 : `INT`, `BIGINT`, `FLOAT`, `DOUBLE`, `DECIMAL`, `BOOLEAN`
    * 문자형 : `CHAR(n)`, `VARCHAR(n)`, `TEXT`
    * 날짜 / 시간형 : `DATE`, `DATETIME`, `TIMESTAMP`, `TIME`, `YEAR`
    * 바이너리 : `BLOB`
* 제약조건 (constraints) [→ 각 제약조건에 대한 자세한 내용 보기](constraint-theory.md)
    * `primary key`
    * `default`
    * `not null`
    * `unique`
    * `check`
    * `foreign key`
* 기타 옵션 및 고려사항
    * `auto_increment`
        * 행이 추가될 때마다 자동으로 1씩 증가하는 숫자 할당
        * 주로 기본키 정수형 컬럼에 사용
    * 인덱스 (`index`)
        * 검색 속도를 빠르게 하기 위한 구조
        * `primary key`와 `unique`는 자동으로 인덱스 생성
    * 테이블 엔진 (`storage engine`)
        * MySQL에서 테이블 데이터를 저장하고 관리하는 방식이나 시스템
        * 데이터가 어떻게 저장, 조회, 수정되는지를 담당하는 백엔드 소프트웨어 컴포넌트
        * MySQL은 대표적으로 `InnoDB`을 사용
        * MyISAM 등도 있음
    * 문자셋 및 정렬법
        * 저장하는 문자 인코딩 방식 지정 (ex. `utf8mb4`)
    * 주석 (`comment`)
        * 테이블이나 컬럼에 대한 설명을 추가할 수 있음
## 테이블 생성문(DDL) 확인하기
* 테이블의 전체 생성 구문을 확인할 수 있음
* 테이블을 복제하거나 구조를 분석할 때 유용
* 테이블 구조를 다른 DB로 옮길 때, DDL 문을 빠르게 복사하여 사용 가능
* 스키마 변경 전후 차이를 비교할 때도 도움이 됨
```sql
[table DDL]

create table t1 (c1 int);
```
```sql
show create table t1;
```
```sql
+-------+----------------------------------------------------------------------------------------------------------------+
| Table | Create Table                                                                                                   |
+-------+----------------------------------------------------------------------------------------------------------------+
| t1    | CREATE TABLE `t1` (
  `c1` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
+-------+----------------------------------------------------------------------------------------------------------------+
1 row in set (0.01 sec)

```
* DEFAULT NULL
    * `not null`을 명시하지 않으면 자동 적용
* ENGINE
    * MySQL 기본 스토리지 엔진(`InnoDB`) 자동 적용
* DEFAULT CHARSET
    * 서버/DB 기본 문자셋 자동 적용
    * 최신 MySQL은 기본 문자셋을 `utf8mb4`로 사용
* COLLATE
    * 문자셋에 맞는 기본 정렬 자동 적용 (`utf8mb4_0900_ai_ci`)
<br>

# 테이블 구조 변경하기 (ALTER)
### 컬럼 추가
* 기본적으로 테이블 맨 마지막 컬럼에 추가
* 위치를 지정하려면 `first / after 컬럼명` 사용
* 컬럼 순서는 논리적 의미만 있을 뿐 성능에는 영향 없음
```sql
alter table 테이블명 add column 컬럼명 자료형 [제약조건] [comment];
```
#### ex)
```sql
create table t1 (c1 int);
alter table t1 add column c2 varchar(10) not null comment '이름';
alter table t1 add column c3 char(5) after c1;

show create table t1;
```
```sql
| t1    | CREATE TABLE `t1` (
  `c1` int DEFAULT NULL,
  `c3` char(5) DEFAULT NULL,
  `c2` varchar(10) NOT NULL COMMENT 'aaa'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
```
* 테이블 t1의 컬럼 순서 : c1 → c3 → c2
### 컬럼 수정
* 자료형 및 컬럼 제약 변경 가능
```sql
alter table 테이블명 modify column 컬럼명 새_자료형 [제약조건];
```
#### ex)
```sql
alter table t1 modify column c1 bigint; -- 자료형 변경
alter table t1 modify column c3 char(5) not null; -- 제약 변경
alter table t1 modify column c2 varchar(5) null comment 'bbb'; -- 자료형, 제약, 주석 변경

show create table t1;
```
```sql
| t1    | CREATE TABLE `t1` (
  `c1` bigint DEFAULT NULL,
  `c3` char(5) NOT NULL,
  `c2` varchar(5) DEFAULT NULL COMMENT 'bbb'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci |
```
#### 이미 데이터가 들어있는데 자료형을 변경하면 어떻게 될까?
* 숫자형 → 문자형
```sql
insert into t1 values (10, 'a', 'abc');
alter table t1 modify column c1 varchar(10); -- success
```
* 문자형 → 숫자형
```sql
alter table t1 modify column c2 int; -- ERROR 1366
```
```sql
ERROR 1366 (HY000): Incorrect integer value: 'abc' for column 'c2' at row 1
```
* 문자형 길이 확대
```sql
alter table t1 modify column c2 char(10); -- success
```
* 문자형 길이 축소
```sql
[t1]
+------+------+------+
| c1   | c3   | c2   |
+------+------+------+
| 10   | a    | abc  |
+------+------+------+

alter table t1 modify column c3 char(3); -- success
alter table t1 modify column c2 char(1); -- ERROR 1265
```
```sql
ERROR 1265 (01000): Data truncated for column 'c2' at row 1
```
* c2에 이미 'abc'라는 데이터가 들어있는데 더 짧은 길이의 데이터 타입으로 변경 시도
    * `strict mode`에 따라 경고 또는 에러 출력
#### strict_mode 확인
```sql
select @@sql_mode;
```
```sql
| ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION |
```
* `STRICT_TRANS_TABLES` 또는 `STRICT_ALL_TABLES` 이면 strict_mode
#### non-strict-mode 로 설정 후 쿼리 실행
* non-strict-mode는 테스트/학습 목적에서만 사용 권장
* 운영 환경에서는 session / global 변경 모두 지양
```sql
set session sql_mode = 'NO_ENGINE_SUBSTITUTION'; -- 현재 sesseion에만 적용 
alter table t1 modify column c2 char(1);
```
```sql
Query OK, 1 row affected, 1 warning (0.02 sec)
Records: 1  Duplicates: 0  Warnings: 1 -- warning 출력
```
#### warning 내용 확인
```sql
show warnings;
```
```sql
+---------+------+-----------------------------------------+
| Level   | Code | Message                                 |
+---------+------+-----------------------------------------+
| Warning | 1265 | Data truncated for column 'c2' at row 1 |
+---------+------+-----------------------------------------+
1 row in set (0.00 sec)
```
#### 데이터 변경 결과 확인
```sql
[ before ]                  [ after ]
+------+------+------+      +------+------+------+
| c1   | c3   | c2   |      | c1   | c3   | c2   |
+------+------+------+  →   +------+------+------+
| 10   | a    | abc  |      | 10   | a    | a    |
+------+------+------+      +------+------+------+
```
* 운영 DB에서는 항상 `strict mode` 권장
### 컬럼 이름만 변경
* 컬럼 이름만 변경
```sql
alter table 테이블명 rename column 기존컬럼명 to 새컬럼명;
```
ex)
```sql
alter table t1 rename column c2 to c2_new;
```
### 컬럼 이름과 정의 변경
* 이름과 컬럼 정의를 동시에 변경할 수 있음
* 자료형을 반드시 다시 명시해야 함
```sql
alter table 테이블명 change column 기존컬럼명 새컬럼명 자료형 [제약조건];
```
#### ex)
```sql
alter table t1 change column c2 new_c2 varchar(50);
alter table t1 change column c3 new_c3; -- ERROR 1064
```
```sql
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near '' at line 1
```
* 자료형을 명시하지 않을 경우 syntax error 출력
### 컬럼 삭제
* 컬럼에 저장된 데이터까지 삭제
* **복구 불가하기 때문에 운영 환경에서는 주의 필요**
    * 즉시 물리 반영되며 rollback 불가
    * MySQL 내부 기능으로는 되돌릴 수 없음 (DDL flashback 기능 없음)
    * `backup`으로는 복구 가능
        * redo/undo log, binlog로도 컬럼 단위 복구는 불가
* **컬럼 삭제 전 반드시 해야 할 것**
    * DDL 리뷰
    * 백업 존재 확인
    * 가능하면 컬럼 삭제 예고(deprecated) 후 삭제
```sql
alter table 테이블명 drop column 컬럼명;
```
#### 안전하게 컬럼 삭제하기
```sql
alter table t1 rename column c3 to c3_deprecated; -- 1단계 : 논리적 삭제
-- 2단계 : 코드 제거 & 모니터링
alter table t1 drop column c3_deprecated; -- 3단계 : 물리적 삭제
select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
| 10   | a    |
+------+------+
1 row in set (0.00 sec)
```
### 제약 조건 추가/삭제
```sql
alter table 테이블명 add constraint 제약이름 제약조건;
alter table 테이블명 drop 제약조건 제약이름;
```
#### ex)
```sql
create table t1 (C1 int primary key);
create table t2 (c1 int, c2 int, c3 int);

alter table t2 add constraint pk primary key(c1); -- pk 추가
alter table t2 add constraint uk unique key(c2); -- uk 추가
alter table t2 add constraint fk foreign key(c3) references t1(c1); -- fk 추가

alter table t2 drop foreign key fk; -- fk 삭제
```
#### pk 삭제하기
* `primary key`는 이름이 있어도 이름 기반 삭제를 허용하지 않음
* `auto_increment` 나 `fk`가 참조중일 경우 먼저 제거한 후 pk 삭제 가능
* pk 삭제 후에는 반드시 새로운 pk를 정의할 것을 권장
```sql
alter table t1 drop primary key;
```
#### uk 삭제하기
* unique key는 내부적으로 인덱스로 구현됨
* `index` 또는 `key`로 삭제해야 함
```sql
alter table t1 drop index uk;
alter table t1 drop key uk;
```
<br>

# 테이블 삭제하기 (DROP)
* 테이블 자체를 완전 삭제
* 테이블 구조, 데이터, 인덱스, 제약조건 모두 제거
```sql
drop table 테이블명;
```
#### ex)
```sql
drop table t1;
```
#### 테이블이 존재할 때만 삭제
* 테이블이 없어도 에러가 발생하지 않음
```sql
drop table if exists t1;
```
<br>

# 테이블 내 모든 데이터 삭제하기 (TRUNCATE)
* 테이블 내 모든 데이터를 빠르게 삭제
* 테이블 구조와 제약조건은 유지
* DML인 `delete`보다 훨씬 빠름
    * 로그 기록 최소화
    * 테이블 초기화 방식
* 대부분 즉시 커밋
* 롤백이 불가하므로 데이터 백업 후 사용 권장
* 트리거 실행 안됨
```sql
truncate table t1;
```
<br>

# 현재 DB의 모든 테이블 조회하기
```sql
show tables;
```
<br>