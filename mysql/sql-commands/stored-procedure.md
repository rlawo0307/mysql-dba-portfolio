```
[참고]

view : select를 저장해둔 가상 테이블
stored procedure : 여러 SQL을 묶어 둔 호출 가능한 프로그램
trigger : 특정 DML 이벤트 발생 시 자동 시행되는 이벤트 기반 로직
```
<br>

# Stored Procedure
* 여러 SQL을 묶어 DB 서버 안에 저장해 두고 필요할 때 호출하는 프로그램
* 변수, 조건문, 반복문, 예외 처리 사용 가능
* 장점
    * 반복 로직 재사용 가능
    * 매번 같은 작업을 애플리케이션에서 길게 보내지 않아도 됨
    * 비즈니스 로직 일부를 DB에서 처리 가능
    * 네트워크 왕복 감소
* 단점
    * 디버깅이 불편함
    * 애플리케이션과 DB로 로직이 분산될 수 있음
    * DB 종속성이 강해질 수 있음
## 변수
* 로컬 변수
    * 프로시저 내부 전용 변수
        * 외부에서 접근 불가
        * 다른 쿼리에서 사용 불가
    * begin ~ end 블록 내부에서만 사용 가능
    * 프로시저가 종료되면 사라짐
    * `declare`로 선언
    * 명시적 타입 지정 필요
* 세션 변수
    * 세션 전체에서 공유하는 변수
    * 세션 연결이 종료되면 사라짐
    * 별도의 선언 없이 사용 가능
    * 접두사(`@`) 필요
```sql
set @a = 100; -- 세션 변수 a 선언 및 초기화

delimiter // -- 구분자 변경
create procedure procedure1() -- 프로시저 생성
begin 
    declare a int; -- 로컬 변수 a 선언
    set a = 10; -- 변수 a에 값 지정
    select a as local_a, @a as session_a;
end //
delimiter ; -- 구분자를 다시 원래대로 복구

call procedure1(); -- 프로시저 호출
```
```sql
local_a | session_a
--------|----------
10      | 100
```
## 매개변수
* `in`
    * 입력 전용
    * 프로시저 안에서 입력값으로만 사용
* `out`
    * 출력 전용
    * 프로시저 안에서 값 설정하며 호출한 쪽의 변수에 그 값이 저장됨
* `inout`
    * 입출력 겸용
    * 호출 시 값을 전달받고 프로시저 내부에서 수정한 뒤 값을 반환
```sql
delimiter //
create procedure procedure1(in in_param int,
                            out out_param int,
                            inout inout_param int)
begin
    select * from t1 where c1 = in_param;
    select count(*) into out_param from t1; -- 쿼리 결과를 변수에 담을 수 있음 
    set inout_param = inout_param + 10;
end //
delimiter ;
```
```sql
set @a = null; -- 세션변수 a 선언 및 초기화
set @b = 20; -- 세션변수 b 선언 및 초기화

call procedure1(1, @a, @b); -- in_param에 1 전달
                            -- @a를 out_param에 연결
                            -- @b를 inout_param에 전달

select @a; -- 테이블 t1의 전체 row 수 출력
select @b; -- 30
```
## 흐름 제어
* 조건문
    * if
    * elseif
    * else
* 반복문
    * loop
    * while
    * repeat
```sql
delimiter //
create procedure procedure1()
begin
    declare a int; -- 로컬 변수 a 선언
--------------------------------------------------------
-- 조건문
--------------------------------------------------------
    set a = 5;

    if a < 0 then
        select 'aaa';
    elseif a < 10 then
        select 'bbb'; -- 'bbb' 출력
    else 
        select 'ccc';
    end if;
--------------------------------------------------------
-- loop
--------------------------------------------------------
    set a = 10;

    loop1 : loop
        if a = 50 then
            leave loop1;
        end if;

        set a = a + 10;
    end loop;
--------------------------------------------------------
-- while
--------------------------------------------------------
    set a = 10;

    while a < 50 do
        set a = a + 10;
    end while;
--------------------------------------------------------
-- repeat
--------------------------------------------------------
    set a = 10;

    repeat
        set a = a + 10;
    until a = 50 -- 조건이 true면 종료
    end repeat;
end //
delimiter ;
```
## 예외 처리
* 저장 프로시저에서 SQL 실행 중 발생하는 에러, 경고, 특수 상태를 처리하는 기능
    * 단순 에러 처리
    * 트랜잭션 롤백
    * 로그 기록
    * 흐름 제어
### 기본 문법
```sql
declare [continue | exit] handler for condition
begin
-- 오류 처리 로직
end;
```
* handler 종류
    * `continue`
        * 에러가 발생한 SQL은 실패
        * 핸들러 실행 후 프로시저의 다음 코드 진행
    * `exit` : 핸들러 실행 후 블록 즉시 종료 
* condition 종류
    * `sqlexception` : 모든 SQL 에러
    * `sqlwarning` : 경고
    * `not found` : 커서에 row가 없는 경우
    * 특정 에러 코드 [→ MySQL 에러 코드 공식 문서](https://dev.mysql.com/doc/mysql-errors/8.0/en/)
### 직접 에러 발생시키기
* signal을 통해 원하는 조건에서 의도적으로 직접 에러를 발생시킬 수 있음
* 별도 권한은 필요 없음
* `45000` : 사용자 정의 에러 코드 
```sql
signal sqlstate '45000'
    set message_text = '사용자 정의 에러';
```
## Cursor
* select 결과 집합에 대해 현재 읽고 있는 위치를 가리키는 포인터
    * 보통 select는 결과 집합을 한 번에 다루는 `set-based` 방식
    * 커서는 그 결과를 `row-by-row`로 처리
* 집합 연산으로 해결하기 어려운 절차적 처리 가능
    * 각 row마다 별도 로직 적용해야 할 때
    * 루프를 돌면서 조건 분기해야 할 때
    * 여러 테이블에 순차적으로 반영해야 할 때
    * 커서 + 핸들러 + 루프로 한 줄씩 처리가 필요할 때
* 커서는 앞으로만 이동 가능
* 커서는 오로지 조회용으로만 사용 가능
    * 커서를 통해 읽은 결과 집합을 커서로 수정할 수 없음
* 엔진이 필요에 따라 결과를 원본 테이블에서 직접 읽을 수도 있고 내부 복사본을 사용할 수 있음
    * 사용자는 어느 방식으로 처리되는지 알 수 없음
* 동작 방식
    1. 커서 선언 → `declare`
    2. 커서 열기 → `open`
        * select 쿼리 실행 시작
        * 결과 집합이 생성됨
        * 내부 포인터가 첫 row 앞에 위치
    3. 한 행씩 읽기 → `fetch`
        * 현재 포인터 위치에서 다음 row를 가져옴
        * 변수에 저장
            * select 컬럼 개수만큼 fetch into 변수 개수가 필요
        * 포인터를 다음으로 이동
    4. 반복 처리 → `loop`
    5. 종료 감지
        * 마지막 row 이후 fetch하면 데이터가 없음
        * `not found` 에러 발생
        * loop 종료 코드 작성 필요
    6. 커서 닫기 → `close`
        * 결과 집합 해제
        * 커서 사용 종료
```sql
declare done boolean default false; -- 종료 감지 변수 선언
declare v1 int; -- fetch된 row를 저장할 변수
declare v2 int;

declare cur1 cursor for select c1, c2 from t1; -- 커서 선언
declare continue handler for not found set done = true;
-- 핸들러 선언
-- 더 이상 읽을 row가 없어서 not found 에러가 발생하면 done 변수 값을 true로 변경

open cur1; -- 커서 오픈

read_loop: loop
    fetch cur1 into v1, v2; -- 한 행씩 가져와 변수 v1, v2에 저장
                            -- c1은 v1에 저장
                            -- c2는 v2에 저장

    if done then -- 종료가 감지되면 loop 탈출
        leave read_loop;
    end if;

    -- 데이터 처리 로직
end loop;

close cur1; -- 커서 사용 종료
```