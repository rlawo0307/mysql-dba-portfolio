# 테이블 생성하기
```sql
create table 테이블명 (
    컬럼명 자료형 [제약조건] [comment],
    ...
) [comment];
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
<br>

# 테이블 생성문(DDL) 확인하기
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
```
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
    * `not null`을 명시하지 않으면 자동으로 적용
* ENGINE
    * MySQL 기본 스토리지 엔진(`InnoDB`) 자동 적용
* DEFAULT CHARSET
    * 서버/DB 기본 문자셋 자동 적용
    * 최신 MySQL은 기본 문자셋을 `utf8mb4`로 사용
* COLLATE
    * 문자셋에 맞는 기본 정렬 자동 적용 (`utf8mb4_0900_ai_ci`)
<br>

# 현재 DB의 모든 테이블 조회하기
```sql
show tables;
```