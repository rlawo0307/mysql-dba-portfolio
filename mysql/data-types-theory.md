# INT vs BIGINT
* `int`
    * 4byte
    * -$2^{31}$ ~ $2^{31}$-1
    * -2,147,483,648 ~ 2,147,483,647
* `bigint`
    * 8byte
    * -$2^{63}$ ~ $2^{63}$-1
    * −9,223,372,036,854,775,808 ~ 9,223,372,036,854,775,807
<br>

### 데이터 범위 비교
#### table 생성
```sql
create table t1 (c1 int, c2 bigint);
```
#### 데이터 삽입
```sql
insert into t1 values (2147483647, 2147483647); -- success
insert into t1 values (2147483648, 2147483648); -- ERROR 1264
```
```sql
ERROR 1264 (22003): Out of range value for column 'c1' at row 1
-- c1 오버플로우 에러 발생
```
<br>

# FLOAT vs DOUBLE
* `float`
    * 근사값(부동소수점)
    * 정밀도 약 7자리
    * 오차가 큼
    * 4byte
* `double`
    * 근사값(부동소수점)
    * 정밀도 약 15~16자리
    * 오차가 작음
    * 8 byte
<br>

### 데이터 오차범위 확인
#### table 생성
```sql
create table t1 (c1 float, c2 double);
```
#### 데이터 입력
```sql
insert into t1 values (0.1+0.2, 0.1+0.2);
```
#### 데이터 확인
```sql
select * from t1;
```
```
+------+------+
| c1   | c2   |
+------+------+
|  0.3 |  0.3 |
+------+------+
1 row in set (0.00 sec)
```
* c1(float) 과 c2(double) 값이 동일한 이유
    * 값이 정확하게 계산된게 아니라, 출력 시 반올림/포맷 때문에 같아 보이는 것
    * MySQL이 `select` 수행 시 보기 좋게 출력해주는 것일 뿐이다
    * 내부적으로는 실제 값이 다름
### 실제 값 확인
```sql
select format(c1, 20) as c1_fmt, format(c2, 20) as c2_fmt from t1;
```
```
+------------------------+------------------------+
| c1_fmt                 | c2_fmt                 |
+------------------------+------------------------+
| 0.30000001192092896000 | 0.30000000000000000000 |
+------------------------+------------------------+
1 row in set (0.00 sec)
```
* c1(float)의 실제 값이 0.3과 오차가 있는 것을 확인할 수 있음
* 하지만 여전히 c2(double)의 오차는 확인 불가
    * 오차가 없는 것이 아니라, `format()` 단계에서 반올림으로 오차를 잘라낸 것
### double의 오차 확인
#### 테이블 생성
```sql
create table t2 (d double);
```
#### 0.1을 반복해서 더하기
```sql
insert into t2(d) select 0.1 from information_schema.columns limit 1000;
```
#### 오차 확인
```sql
select sum(d) as sum_double, sum(d) - 73.7 as double_diff from t2;
```
```
+-------------------+---------------------------------+
| sum_double        | double_diff                     |
+-------------------+---------------------------------+
| 73.70000000000009 | 0.00000000000008526512829121202 |
+-------------------+---------------------------------+
```
* `sum_double`은 정확히 73.7이 아님
* 즉, double의 누적 연산 결과는 수학적 값과 오차가 있음이 증명됨
<br>

# DECIMAL
* 정확값(고정소수점)
* 형식 : `decimal`(전체 자릿수, 소수점 이하 자릿수)
    * 전체 자리수 : `n`
    * 소수점 이하 자릿수 : `m`
    * 정수부 자릿수 : `n-m`
<br>

### 범위를 초과하는 데이터를 삽입할 경우
* 소수점 범위를 초과한 경우, 소수점 `m+1` 번째 자리에서 반올림한 후 범위 검사 수행
* 정수부 자릿수가 초과한 경우 반올림 검사 단계로 넘어가지 않고 바로 에러 출력 → `ERROR 1264`
#### 테이블 생성
```sql
create table t1 (c1 decimal(5, 2));
```
#### 데이터 삽입
```sql
insert into t1 values (99.9); -- success
insert into t1 values (99.99); -- success
insert into t1 values (999.99); -- success
insert into t1 values (99.999); -- success
insert into t1 values (99.9999); -- seccess
insert into t1 values (9999.99); -- ERROR 1264
```
#### 결과 조회
```sql
select * from t1;
```
```
+--------+
| c1     |
+--------+
|  99.90 | -- 무조건 소수점 자릿수를 채워서 저장
|  99.99 |
| 999.99 |
| 100.00 | -- 소수점 3번째에서 반올림 후 범위 검사
| 100.00 | -- 소수점 3번째에서 반올림 후 범위 검사
+--------+
5 rows in set (0.00 sec)
```
<br>

### DECIMAL 끼리의 연산
* 덧셈, 뺄셈
    * 정수부 범위 계산 시 carry(자리 올림)를 고려하여 +1
    * 전체 자릿 수 계산 시 carry를 고려하여 +1
```sql
decimal(p1, s1) + decimal(p2, s2) = decimal(p3, s3) 의 경우

s3 = max(s1, s2)
p3 = s3 + max(p1-s1, p2-s2) + 2

ex)
decimal(5, 2) + decimal(4, 1) = decimal(7, 2)
decimal(7, 3) - decimal(10, 4) = decimal(12, 4)
```
* 곱셈
    * 전체 자릿 수 계산 시 carry를 고려하여 +1
```sql
decimal(p1, s1) * decimal(p2, s2) = decimal(p3, s3) 의 경우

s3 = s1 + s2
p3 = p1 + p2 + 1

ex)
decimal(5, 2) * decimal(4, 3) = decimal(10, 5)
```
* 나눗셈
    * s3를 구할 때 사용한 숫자 `6`은 decimal 나눗셈 결과의 최소 소수점 자리수를 의미
        * 너무 짧으면 의미 없는 결과가 나오고
        * 너무 길면 성능/메모리 문제가 있으므로
        * 의미 있는 최소 정밀도로 `6`을 선택
        * 즉, 나눗셈의 결과는 최소 소수점 6자리를 보장함
```sql
decimal(p1, s1) / decimal(p2, s2) = decimal(p3, s3) 의 경우

s3 = max(6, s1 + p2 + 1) -- 6 = 정밀도 하한선
p3 = p1 - s1 + s3

ex)
decimal(5, 2) / decimal(4, 1) = decimal(10, 7)
```
<br>

### DECIMAL의 혼합연산
* MySQL 타입 우선순위
    * `double`, `float`
    * `decimal`
    * `int`, `bigint`
    * 더 위에 있는 타입으로 암묵적 변환
* `int`와의 연산
    * decimal과 int의 연산결과는 무조건 `decimal`
    * n자리 int는 내부적으로 `decimal(n, 0)`으로 취급
* `double`과의 연산
    * decimal과 double의 연산결과는 무조건 `double`
    * 즉, decimal의 정확성은 즉시 파괴
    * decimal의 정확성을 지키고 싶다면 double을 decimal로 캐스팅 필요    
<br>

# CHAR vs VARCHAR
* `char(n)`
    * 고정 길이
    * 지정한 길이만큼 항상 채움
    * 부족한 길이만큼 공백 자동 패딩
    * 디스크/메모리 위치 계산이 단순
    * 비교 시 char 컬럼 값의 trailing space 제거 후 비교
* `varchar(n)`
    * 가변 길이
    * 실제 길이만큼만 저장
    * 공백 그대로 유지
    * 실제 데이터 길이 + 길이 정보(1~2byte) 저장
        * `(n)`은 문자 개수 제한, 저장 공간 제한이 아님
    * 비교 시 공백 포함 (공백을 데이터로 취급)
### 길이 비교해보기
* `length()`는 문자열의 바이트 길이 반환
    * char 타입의 경우 뒤 공백을 제거한 후 계산
* **MySQL은 SQL 레벨에서 char 값을 다룰 때 항상 뒤 공백을 제거한 후 보여줌**
```sql
create table t1 (c1 char(10), c2 varchar(10));
insert into t1 values ('aaa', 'aaa');
insert into t1 values ('bbb', 'bbb ');
select c1, c2, length(c1) as char_length, length(c2) as varchar_length from t1;
```
```sql
+------+------+-------------+----------------+
| c1   | c2   | char_length | varchar_length |
+------+------+-------------+----------------+
| aaa  | aaa  |           3 |              3 | -- 값이 같은 이유? length() 때문에
| bbb  | bbb  |           3 |              4 |
+------+------+-------------+----------------+
2 rows in set (0.00 sec)
```
### char 타입이 왜 필요할까?
* char의 패딩 공백은 SQL 레벨에서 값이 아니라, **저장, 비교, 인덱스 구조에서 차이를 만든다**
#### 비교 연산에서의 char
* 의미적으로 동일해야 하는 코드에 사용
* 공백 차이를 무시해야 하는 경우 사용
```sql
create table t1 (c1 char(10), idx int);
insert into t1 values ('abc', 1), ('abc ', 2);
select * from t1 where c1 = 'abc';
```
```
+------+------+
| c1   | idx  |
+------+------+
| abc  |    1 |
| abc  |    2 |
+------+------+
2 rows in set (0.01 sec)
```
* 참고) `'abc '` 와 비교할 경우
    * `'abc '` 는 SQL 문자열 리터럴이며 trailing space가 제거되지 않음
    * 비교 시 char 컬럼만 trailing space가 제거됨
```sql
select * from t1 where c1 = 'abc ';
Empty set (0.00 sec)
```
#### UNIQUE, PK 제약에서의 char
* UNIQUE/PK에서 공백 중복을 막고 싶을 때 사용
    * UNIQUE 와 PK 제약이 걸려있는 컬럼의 값은 테에블 내에서 유일해야 함
    * char 타입 컬럼 값은 패딩 공백 제거 후 같은 문자열이면 동일한 값으로 인식
        * **저장 값이 같은게 아니라 비교 규칙에서 trailing space를 제거하기 때문에**
```sql
create table t1 (c1 char(10) unique);
insert into t1 values ('a'); -- success
insert into t1 values ('a '); -- ERROR 1062
```
```sql
ERROR 1062 (23000): Duplicate entry 'a' for key 't1.c1'
```
#### 인덱스 구조에서의 char
* char 인덱스
    * 고정 길이이므로 B-Tree 노드에서 정렬 계산이 단순해 질 수 있음
    * 예전 MyISAM 시대에서 varchar 인덱스와 성능 차이가 있었음
    * **InnoDB에서의 거의 체감 불가**
    * 현재는 인덱스 키의 길이가 정말 같을때만 char 사용
#### 저장 구조에서의 char
* char
    * 고정 길이로 row 내 위치가 고정적
    * 메모리 접근이 단순
* varchar
    * 가변 길이로, 길이 변경 시 row 재배치가 발생할 수 있음
* **즉, update가 자주 일어나는 테이블이라면 char가 이론적으로 유리할 수 있음**
<br>

