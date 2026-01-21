# EXPLAIN이란?
* 쿼리 실행 계획을 보여주는 도구
* MySQL 옵티마이저가 쿼리를 어떻게 실행할지 예상한 계획을 보여줌
* 테이블 접근 순서, 인덱스 사용 여부, 예상 읽는 행(row) 수 등을 알 수 있음
* 쿼리를 실제로 실행하지 않고 실행 계획만 확인할 수 있음
* 종류
    * `EXPAIN`
        * 기본 실행 계획 확인
    * `EXPLAIN ANALYZE`
        * 실제 실행까지 하면서 자세한 실행 시간과 행 처리 정보 제공
    * `EXPLAIN FORMAT=JSON`
        * JSON 포맷으로 상세한 실행 계획 확인
<br>

# EXPLAIN 결과 컬럼
* id
    * 쿼리 내 `select` 단위의 실행 순서
    * 보통 값이 작을수록 먼저 실행
    * 단일 쿼리는 보통 `1`
* select_type
    * select의 종류
        * `SIMPLE` : 서브쿼리 없는 단순 select
        * `PRIMARY` : 최상위 select
        * `SUBQUERY` : where 절 안의 서브쿼리
        * `DERIVED` : from 절의 서브쿼리 (임시 테이블)
        * `UNION` : union의 두 번째 이후 select
* table
    * 접근 대상 테이블 명
    * 서브쿼리나 derived table이면 별칭으로 표시
* partitions
    * 접근하는 파티션 정보
    * 파티션 테이블이 아니면 `NULL`
    * 파티션 pruning 여부 확인용
* type
    * 테이블 접근 방식 (아래로 내려갈 수록 성능 좋음)
        * `ALL` : Full Table Scan
        * `index` : Index Full Scan
        * `range` : Index Range Scan
        * `ref` : Non Unique Index + `=` 조건 (여러 row 가능)
        * `eq_ref` : PK/Unique Index + `=` 조건 (조인에서만 등장, row 1개 보장)
        * `const` : PK/Unique Index + 상수 비교
        * `system` : 테이블에 row가 딱 1개만 존재
* possible_keys
    * 사용 가능하다고 판단한 인덱스 목록
    * 실제로 사용하지 않아도 표시될 수 있음
* key
    * 실제로 사용한 인덱스
    * `NULL`이면 인덱스 미사용
* key_len
    * 사용한 인덱스의 byte 길이
    * 복합 인덱스에서 어떤 컬럼까지 사용했는지 추적 가능
    * 컬럼이 `null`을 허용하는 경우, null 플래그(1 byte) 추가
        * `not null`인 경우 추가되지 않음
* ref
    * 인덱스 비교 대상 값
    * 주로 조인 조건에서 의미 있음
    * `const` 일 경우, 상수와 비교
* rows
    * 옵티마이저가 읽을 것으로 예상한 row 수
    * 실제 결과 row수 아님
    * 성능 튜닝 시, 인덱스 전/후 `rows` 감소가 핵심 지표가 됨
* filtered
    * where 조건을 통과할 것으로 옵티마이저가 예상한 row 비율 (%)
    * 통계 기반의 추정값이므로 절대값 자체는 신뢰하기 어려움
    * **값의 크기보다 변화 추세를 해석하는 것이 중요**
    * `rows * filtered / 100 = 결과 row 추정`
* Extra
    * 부가적인 실행 정보
        * `Using where` : `where`절 조건을 필터링함
        * `Using index` : covering index
        * `Using filesort` : MySQL이 임시로 정렬 작업 수행
        * `Using temporary` : 임시 테이블 사용
        * `Using index condition` : ICP 적용
        * `Backward index scan` : 인덱스 역순 스캔
<br>