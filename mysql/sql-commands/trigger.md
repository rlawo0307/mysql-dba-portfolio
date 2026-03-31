```
[참고]

view : select를 저장해둔 가상 테이블
stored procedure : 여러 SQL을 묶어 둔 호출 가능한 프로그램
trigger : 특정 DML 이벤트 발생 시 자동 시행되는 이벤트 기반 로직
```
<br>

# Trigger
* 특정 DML 이벤트가 발생했을 때 자동으로 실행되는 SQL 코드
* 애플리케이션 로직 없이 DB 레벨에서 자동 처리
* 데이터 변경이 발생할 경우 트리거 자동 실행 (명시적 호출 불가)
* 최초로 생성되면 테이블에 종속됨 = 특정 테이블에 묶인 자동 실행 코드
* 트리거 내 제한 사항
    * 본인 테이블 직접 수정 불가
        * 간접적인 트리거 연쇄 실행이 발생할 수 있음
        * 에러 발생
    * commit / rollback 불가
        * 트리거는 독립 실행이 아닌 DML의 일부
        * 트리거는 이미 외부 트랜잭션에 포함됨
        * 트리거 실패 시 트랜잭션 전체가 rollback 됨
        * 트리거 내부에서 수행된 작업 또한 함께 rollback 됨
    * 결과 반환 불가
* 장점
    * 데이터 무결성 보장
    * 로그 기록을 남길 수 있음
    * 파생 데이터를 자동으로 업데이트 할 수 있음
* 단점
    * DML마다 실행되므로 과도하면 성능 저하를 유발할 수 있음
    * 디버깅 어려움
    * 간접적인 트리거 연쇄 실행이 발생할 수 있음
## OLD / NEW
* 하나의 row를 기준으로 엔진이 트리거 실행 시점에 제공하는 가상의 레코드
    * `old`
        * 변경 이전 row
        * 트리거 실행 시점에서의 기존 값
    * `new`
        * 변경 이후 row 또는 입력될 row
        * 트랜잭션 내에서 생성된 새로운 값
### 이벤트별 사용 가능 여부
* `O` = 사용 가능
* `X` = 사용 불가

|          |old|의미|new|의미|비고|
|----------|:-:|:-:|:-:|:--:|:-:|
|**insert**|X|-|O|입력될 row|before trigger에서 값 수정 가능|
|**update**|O|변경 이전 row|O|변경 이후 row|변경 전후 값 비교 가능|
|**delete**|O|삭제될 row|X|-|-|

## 문법
```sql
create trigger trigger1 {before | after} {insert | update | delete} on t1
for each row
begin
-- 실행 로직
end;
```
* 트리거 실행 시점
    * `before`
        * 데이터 변경 직전에 실행
        * `new` 값 수정 가능
    * `after`
        * 데이터 변경 후에 실행
        * new 수정 불가
        * 결과 기반 처리 (로그 기록, 파생 데이터 업데이트 등)
* `for each row`
    * 각 row마다 트리거 실행
    * row 단위라서 성능에 영향을 끼침
    * MySQL은 statement-level trigger 없음
* `begin ... end`
    * 트리거 내부 실행 블록
    * 여러 SQL 실행 가능
## 동작
* 모든 단계는 동일한 트랜잭션 내에서 실행됨
```text
1. DML 실행 시도
2. before trigger 실행
3. 실제 데이터 변경
4. after trigger 실행
5. commit 또는 rollback
```
# Trigger와 Deadlock
* deadlock 발생 원인 [→ deadlock에 대한 내용 자세히 보기](../transaction/deadlock.md)
    * 트리거는 자동 실행되므로 사용자가 트리거 내부 동작을 바로 알기 어려움
        * 사용자가 예상한 것보다 더 많은 테이블에 대한 lock을 획득할 수 있음
        * 사용자가 작성한 SQL만으로는 실제 lock 획득 순서를 알기 어려움
        * 예상하지 못한 deadlock이 발생할 수 있음
    * 트리거는 별도 커밋을 하지 않음
        * 트리거 내부 작업이 많을수록 lock 보유 시간이 길어짐
        * 다른 트랜잭션의 대기 시간이 길어짐
        * 트랜잭션 간 충돌 가능성 증가
        * deadlock 발생 확률 증가
    * row-level trigger라서 반복 실행됨
        * 각 row마다 트리거 실행
        * 추가적인 lock이 반복적으로 발생할 수 있음
        * lock이 많이 생길수록 deadlock 확률도 올라감
    * 트리거로 인해 트랜잭션 간 lock 획득 순서가 충돌할 수 있음
        * Transaction A : t1 → trigger → t2
        * Transaction B : t2 → trigger → t1 
### 대책
* lock 획득 순서를 일관되게 맞추기
* 트리거 안에서 다른 테이블에 대한 update 최소화하기
* 대량 DML 피하기
    * 범위를 나눠서 처리하거나
    * 트리거를 쓰지 않는 설계 고려
* 트랜잭션을 짧게 유지하기
* 여러 테이블을 연쇄 수정하는 로직 피하기
* deadlock 발생을 전제로 재시도(retry) 로직 설계하기
* **트리거 로직은 명확하게 문서화하기**