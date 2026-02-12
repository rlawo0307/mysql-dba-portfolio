# MySQL Built-in Functions (내장 함수)
### [→ MySQL 내장 함수 공식 문서](https://dev.mysql.com/doc/refman/8.0/en/functions.html)
* MySQL 서버가 미리 구현해 둔 함수
* SQL 실행 과정 중 MySQL 서버 내부에서 평가됨
    * optimizer는 함수의 실제 row 단위 계산은 수행하지 않음
        * constant expression은 미리 계산할 수 있음
    * 내장 함수의 성질을 기준으로 실행 계획을 세움
        * 결정성(`determinisitc`)
            * 같은 입력이 주어지면 항상 같은 결과가 나오는 성질
        * 평가 시점 (쿼리 시작 시 / row 마다)
        * 인덱스 사용 가능성 등
    * 실제 함수 계산(evaluation)은 executor 단계에서 수행
* 별도 정의 없이 바로 사용 가능
## 내장 함수를 사용할 수 없는 위치
### 식별자가 와야 하는 자리
* 식이 아닌 식별자가 와야 하는 자리에는 내장 함수를 사용할 수 없음
* 객체 식별자는 파싱(parser) 및 해석(resolve) 단계에서 메타데이터와 매핑되어야 함
* 내장함수는 실행(executor) 단계에서 평가됨
* 예시
    * 스키마, 데이터베이스, 테이블, 컬럼, 인덱스 이름 등
    * 제약 조건 이름
    * 파티션 이름
    * 트리거, 프로시저, 이벤트 이름
    * 사용자, 역할 이름
    * 별칭(alias) 정의 위치
    * DDL 내부 컬럼 정의 위치 등
```sql
use concat('db', 1); -- X 
select * from concat('t', 1);  -- X 
select concat('c', 1) from t1; -- O (표현식 위치이므로 가능)
select t1.concat('c', 1) from t1; -- X 
create index concat('idx', 1) on t1(c1); -- X 
create table t1 (concat('c', 1) int); -- X 
```
### literal만 허용되는 문법 위치
* literal = 계산 없는 값
    * `expression ⊃ constant expression ⊃ literal`
* 식(expression)처럼 보이지만 특정 리터럴만 허용되는 위치(`literal-only`)에는 내장함수를 사용할 수 없음
    * 내장함수는 function_call expression 형태로 실행됨
    * literal-only 위치는 expression 자체를 허용하지 않음
    * 따라서 함수 호출은 문법 단계(parser)에서 거부됨
* 예시
    * enum / set 정의
    * auto_increment 값
    * 일부 DDL 옵션 값
    * partition 정의
```sql
create table t1 (status enum(upper('A'), 'B')); -- X
alter table t1 auto_increment = abs(10); -- X
create table t1 (c1 int) engine = concat('inno', 'DB') -- X
create table t1 (c1 int) partition by range (c1) (
    partition p0 values less than (abs(10)) -- X
);
```
### constant expression만 허용되는 문법 위치
* constant expression = 쿼리 시작 시 한번 계산되면 고정되는 계산식
    * `expression ⊃ constant expression ⊃ literal`
* row와 무관한 constant expression 허용 위치에서는 deterministic 내장 함수는 허용될 수 있음
* 예시
    * default 값
```sql
create table t1 (c1 int default(abs(10))); -- O
```