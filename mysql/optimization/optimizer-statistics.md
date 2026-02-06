# Optimizer란?
* SQL을 어떻게 실행할지 결정하는 MySQL의 핵심 컴포넌트
* MySQL의 InnoDB는 `Cost-Based-Optimizer(CBO)` 사용
* 여러 실행 계획 후보 중 가장 cost가 낮다고 판단되는 계획을 선택
* scan 방식, join 순서, join 방식, 조건 평가 순서 등 쿼리 실행 전체 전략을 결정
* 동일한 결과를 보장하는 SQL이라도 실행 계획에 따라 성능이 달라질 수 있음
* 옵티마이저의 비용 계산은 **통계 정보를 기반으로 한 추정값**이며, 실제 실행 전에는 정확한 수치를 알 수 없음
## 옵티마이저의 비용 계산 기준
* 예상 읽기 row 수
    * selectivity (선택도)
    * cardinality (카디널리티)
* I/O 비용
    * 디스크 페이지 접근 횟수
    * 테이블 페이지 vs 인덱스 페이지
* CPU 비용
    * 조건 비교
    * 정렬
    * 집계 연산 등
* Back Lookup 비용
# 통계 정보란?
* 테이블과 인덱스의 데이터 분포를 요약해 둔 메타데이터
* 옵티마이저가 cost를 계산할 때 쓰는 유일한 근거 자료
## 통계 정보 조회
### 테이블 통계 정보 → `mysql.innodb_table_stats`
```sql
desc mysql.innodb_table_stats;
```
```sql
+--------------------------+-----------------+------+-----+-------------------+-----------------------------------------------+
| Field                    | Type            | Null | Key | Default           | Extra                                         |
+--------------------------+-----------------+------+-----+-------------------+-----------------------------------------------+
| database_name            | varchar(64)     | NO   | PRI | NULL              |                                               |
| table_name               | varchar(199)    | NO   | PRI | NULL              |                                               |
| last_update              | timestamp       | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED on update CURRENT_TIMESTAMP |
| n_rows                   | bigint unsigned | NO   |     | NULL              |                                               |
| clustered_index_size     | bigint unsigned | NO   |     | NULL              |                                               |
| sum_of_other_index_sizes | bigint unsigned | NO   |     | NULL              |                                               |
+--------------------------+-----------------+------+-----+-------------------+-----------------------------------------------+
```
* `database_name`
    * 통계 정보가 속한 스키마(DB) 이름
* `table_name`
    * 통계 정보가 속한 테이블 이름
    * 내부적으로 살제 테이블과 1:1 매칭
    * 테이블이 drop되면 이 행도 같이 제거됨
    * 하나의 테이블 통계는 `(database_name, table_name)` 조합으로 식별됨
* `last_update`
    * 이 통계 정보가 마지막으로 갱신된 시점
    * 통계 정보가 갱신되는 경우
        * `analyze table`
        * InnoDB 자동 통계 갱신
        * 테이블 생성 직후
    * 갱신 시점이 오래됐을 경우 통계가 현실과 다를 가능성이 높아짐
* `n_rows`
    * 테이블에 존재하는 전체 row 수 **(추정값)**
    * 전체를 count하지 않고 테이터 페이지를 **sampling**해서 추정
    * 모든 cost 계산의 기준값
        * `예상 결과 row 수 = n_rows * selectivity`
    * 실제 row 수와 크게 다르면 옵티마이저의 판단이 틀어질 가능성이 높아짐
    * 데이터 분포가 치우친 경우 추정 오차가 커질 수 있음
* `clustered_index_size`
    * `clustered index(PK)`의 페이지 수
        * InnoDB에서 `테이블 데이터 = PK 인덱스` 이므로
        * 테이블 전체 크기를 의미
    * 1 page = 16KB
        * `실제 크기 = clustered_index_size * 16KB`
    * full table scan 의 cost를 계산할 때 사용
    * PK 기반 range scan cost를 계산할 때 사용
* `sum_of_other_index_sizes`
    * secondary index의 전체 페이지 수의 합
        * unique 인덱스
        * 일반 index
        * PK 제외
    * 보조 인덱스 탐색 비용을 계산할 때 사용
    * back lookup 비용 추정 시 참고
    * 이 값이 크면 인덱스가 많거나 인덱스가 비대함
        * 쓰기 성능이 저하되고 캐시 압박 가능성이 높아짐
### 인덱스 통계 정보 → `mysql.innodb_index_stats`
```sql
desc mysql.innodb_index_stats;
```
```sql
+------------------+-----------------+------+-----+-------------------+-----------------------------------------------+
| Field            | Type            | Null | Key | Default           | Extra                                         |
+------------------+-----------------+------+-----+-------------------+-----------------------------------------------+
| database_name    | varchar(64)     | NO   | PRI | NULL              |                                               |
| table_name       | varchar(199)    | NO   | PRI | NULL              |                                               |
| index_name       | varchar(64)     | NO   | PRI | NULL              |                                               |
| last_update      | timestamp       | NO   |     | CURRENT_TIMESTAMP | DEFAULT_GENERATED on update CURRENT_TIMESTAMP |
| stat_name        | varchar(64)     | NO   | PRI | NULL              |                                               |
| stat_value       | bigint unsigned | NO   |     | NULL              |                                               |
| sample_size      | bigint unsigned | YES  |     | NULL              |                                               |
| stat_description | varchar(1024)   | NO   |     | NULL              |                                               |
+------------------+-----------------+------+-----+-------------------+-----------------------------------------------+
```
* `index_name`
    * 통계 정보가 속한 인덱스 이름
* stat_name
    * `n_diff_pfx`
        * 인덱스를 구성하는 컬럼들을 왼쪽부터 사용했을 때의 cardinality
        * 단일/복합 인덱스 모두에 적용되는 통계 정보
            * 단일 인덱스 (a) : n_diff_pfx01
            * 복합 인덱스 (a, b) : n_diff_pfx01, n_diff_pfx02
            * 복합 인덱스 (a, b, c, d) : n_diff_pfx01 ~ 04
        * `size`
            * 인덱스 전체 페이지 수
            * PK, 보조 인덱스 모두 포함
        * `leaf_pages`, `n_leaf_pages`
            * leaf 페이지 수
* `sample_size`
    * 통계 수집 시 샘플링에 사용된 페이지 수
    * null 가능
    * 값이 작을수록 오차 가능성이 높음
    * `cardinality`가 이상하다면 `sample_size`가 너무 작은지 확인 필요
## Cardinality와 Selectivity의 관계
* Cardinality
    * 인덱스 컬럼에서 서로 다른 값의 개수
    * `cardinality`가 높을 수록 일반적으로 조건에 의해 필터링되는 row 수가 적어짐
    * 옵티마이저는 `cardinality`가 높은 컬럼을 포함한 인덱스를 우선적으로 고려
* Selectivity
    * 조건이 전체 row 중 얼마나 걸러내는지의 비율
    * `selectivity = 조건 결과 row 수 / 전체 row 수`
    * `selectivity`가 낮을수록 cost가 낮다고 판단되는 경향이 있음
    * 옵티마이저는 `selectivity`가 높은 조건을 먼저 적용했을 때 cost가 낮다고 판단하는 경향이 있음
### Cardinality와 Selectivity의 수학적 관계
* 동등(=) 조건일 경우
    * 하나의 값만 선택되므로 cardinality가 클수록 선택되는 row수가 적어짐
    * **selectivity는 cardinality에 반비례**
    * 단, 데이터가 균등 분포라는 전제 하에서만 성립
    * 데이터 분포가 치우쳐 있을 경우 이 추정값은 부정확할 수 있음
    * 이러한 한계를 보완하기 위해 `Histogram`이 사용됨
* 범위 조건일 경우
    * 값의 개수(cardinality) 보다 조건이 전체 데이터 분포 중 얼마나 넓은 구간을 덮는지가 중요
    * cardinality가 높아도 범위가 넓으면 많은 row가 선택될 수 있음
    * 즉, **범위 조건에서는 cardinality 만으로는 selectivity 를 수학적으로 추정할 수 없음**
    * 이를 보완하기 위해 `Histogram`이 사용됨
# Histogram이란?
* 컬럼 값의 분포를 구간별로 저장한 통계 정보
* 컬럼 값을 여러 구간(`bucket`)으로 나누고 각 구간에 몇 개의 row가 있는지 저장
* 이를 바탕으로 옵티마이저가 범위 조건의 `selectivity`를 직접 추정 가능
* 컬럼 단위 통계 정보이며, 인덱스 유무와 무관함
    * 인덱스가 없는 컬럼의 범위 조건에서도 selectivity 추정에 사용됨
* 히스토그램이 cardinality 보다 우선적으로 사용됨
    * 옵티마이저는 빠른 추정이 아니라 **틀리지 않는 추정**을 우선으로 함
    * 히스토그램은 이미 계산되어 있는 정확한 통계이며
    * cardinality와 비교했을 때 조회 비용 차이가 거의 없음
* 기본적으로 자동으로 생성되지 않음
* 종류
    * `Singleton Histogram`
        * 각 값 하나하나를 개별 bucket으로 관리하는 히스토그램
        * 각 값의 정확한 빈도를 알고 있음
        * 사용되는 경우
            * 중복이 거의 없는 컬럼에 사용
        * 장점
            * `=` 또는 `in` 조건에서 매우 정확
        * 단점
            * 값 종류가 너무 많으면 메모리에 부담이 갈 수 있음
            * 정말 필요한 경우에만 사용하는 것을 권장
    * `Equi-Height Histogram`
        * 각 bucket이 비슷한 개수의 row를 가지도록 값을 구간으로 나눈 히스토그램
        * MySQL에서 가장 일반적으로 쓰임
        * 사용되는 경우
            * 범위 조건이 자주 쓰이는 컬럼
            * 값이 연속적이거나 순서가 중요한 경우
        * 장점
            * 범위 조건의 selectivity 추정 정확도 높음
            * 데이터가 한쪽으로 치우져있어도 대응 가능
        * 단점
            * 개별 값 정확도는 singleton보다 떨어짐
            * bucket 개수에 따라 정밀도가 달라짐
* 히스토그램의 효과가 적은 경우
    * 데이터 분포가 이미 균등한 경우
        * cardinality 기반 추정이 실제 분포와 거의 일치함
        * 히스토그램이 보정할 오차가 없음
    * row 수가 매우 적은 테이블
        * full table scan 비용이 낮아 정확한 추정보다 단순한 실행이 더 빠름
    * 조건이 항상 PK/unique lookup인 경우
        * 결과 row 수가 0 또는 1 로 고정
        * 데이터 분포 정보가 실행 계획 결정에 영향을 주지 않음
* 히스토그램의 한계
    * bucket 수가 제한되어 있어 정밀도에 한계가 있음
    * 데이터 변경이 빈번한 테이블에서는 통계 갱신 시점의 통계 정보와 실제 분포 사이의 오차가 생길 수 있음
    * 컬럼 단위 통계이므로 컬럼 간 상관관계는 알 수 없음
        * 여러 조건이 동시에 사용될 경우 selectivity를 단순 곱셉으로 계산
        * 실제 데이터 상관관계가 강할 경우 추정 오차가 발생할 수 있음
### 히스토그램 생성
```sql
analyze table t1 update histogram on c1, c2 with 100 buckets;
```
```sql
+--------+-----------+----------+-----------------------------------------------+
| Table  | Op        | Msg_type | Msg_text                                      |
+--------+-----------+----------+-----------------------------------------------+
| db1.t1 | histogram | status   | Histogram statistics created for column 'c1'. |
+--------+-----------+----------+-----------------------------------------------+
1 row in set (13.48 sec)
```
* `c2`에 대한 통계 수집은 스킵됨
* 컬럼 `c1, c2`에 대해 히스토그램 생성하도록 수행했지만 실제로는 `c1`에 대해서만 생성됨
* MySQL은 의미 없는 히스토그램을 만들지 않음
* 즉, 히스토그램은 지정한 컬럼 모두에 대해 생성되지 않을 수 있음
    * 인덱스 cardinality가 매우 높아 selectivity 추정이 이미 정확한 경우
    * 히스토그램을 지원하지 않는 타입인 경우 (blob, text, json 등)
    * PK 또는 unique 컬럼 등 결과 row 수가 고정된 경우
### 히스토그램 생성 여부 확인
```sql
select schema_name, table_name, column_name, histogram
from information_schema.column_statistics
where table_name = 't1';
```
