# MySQL Built-in Functions (내장 함수)
* MySQL 서버가 미리 구현해 둔 함수
* SQL 실행 과정 중 MySQL 서버 내부에서 평가됨
    * optimizer는 내장 함수를 실행하지 않고 내장 함수의 성질을 기준으로 실행 계획을 세움
        * 결정성 (determinisitc)
        * 평가 시점 (쿼리 시작 시 / row 마다)
        * 인덱스 사용 가능성 등
    * 실제 함수 계산(evaluation)은 executor 단계에서 수행
* 별도 정의 없이 바로 사용 가능
* 내장 함수 분류
    * 문자열 함수
    * 숫자 함수
    * 날짜/시간 함수
    * 집계 함수
    * 윈도우 하뭇
    * 제어 흐름 함수
    * null 처리 함수
    * 타입 변환 함수
    * json 함수
    * 암호화 / 보안 함수
    * 정보 함수
    * 시스템 함수
    * 기타 특수 함수
## 내장 함수를 사용할 수 없는 위치
### 테이블, 컬럼, 인덱스 이름
* 식(expression)이 아니라 식별자(identifier)가 와야 하는 자리
* 이 단계는 파싱 단계로, 객체 이름은 실행 전에 확정 되어야 함
* 내장 함수는 executor 에서 평가되므로 parser에서는 객체의 이름을 알 수 없음
```sql
select * from upper(t1); -- 불가
select * from t1 as LOWER; -- 불가
```