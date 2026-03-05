# 목차
* 테이블 접근 방식
    * Full Table Scan
    * Index Lookup
    * Index Full Scan
    * Index Range Scan
* 인덱스 접근 방식
    * Covering index
    * Back Lookup
* 고급 인덱스 스캔
    * Index Skip Scan
    * Loose Index Scan
* 참고 자료
    * [→ explain에 대한 자세한 내용 보기](../optimization/explain.md)
    * [→ index 종류에 대한 자세한 내용 보기](../index-basics.md)
    * [→ 각 scan 방식에 대한 개념 자세히 보기](../index-basics.md)
    * [→ selectivity 및 통계정보에 대한 개념 자세히 보기](../../optimization/optimizer-statistics.md)
<br>

# 테이블 접근 방식
## Full Table Scan
```sql
create table t1(c1 int, c2 int);
explain select * from t1 where c1 = 10;
```
```sql
type            : ALL -- full table scan
possible_keys   : NULL -- 사용 가능한 인덱스 없음
key             : NULL -- 인덱스 미사용
key_len         : NULL
Extra           : Using where
```
## Index Lookup
* `type = range / ref / eq_ref / const` 등
```sql
create table t1(c1 int, c2 int, index idx1(c1));
explain select * from t1 where c1 = 10;
```
```sql
type            : ref --  Non Unique Index + `=` 조건
possible_keys   : idx1 -- 사용 가능한 인덱스 목록 : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5 -- int(4 bytes) + NULL 플래그(1 byte)
Extra           : NULL
```
## Index Full Scan
```sql
create table t1(c1 int, index idx1(c1));
explain select * from t1;
```
```sql
type            : index -- index full scan
possible_keys   : NULL -- where 조건이 없어서 후보 인덱스를 고려할 필요 없음
key             : idx1 -- 인덱스 사용
key_len         : 5
Extra           : Using index -- covering index 사용
```
## Index Range Scan
```sql
create table t1(c1 int, c2 int, index idx1(c1));
explain select * from t1 where c1 between 10 and 20;
```
```sql
type            : range -- index range scan
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5
Extra           : Using index condition -- where 조건이 인덱스 스캔 단계에서 처리됨
```
<br>

# 인덱스 접근 방식
## Back Lookup
* expain 결과만 보고 back lookup 발생 여부를 확정할 수 없음
* back lookup은 index lookup의 내부 동작이라 따로 표시되지 않음
* 단, 거의 확실하게 추론 가능
    * back lookup이 발생하지 않는 경우
        * covering index로 조건을 처리한 경우 → `Extra=Using index`
    * back lookup이 발생하는 경우
        * 아래 조건이 동시에 만족한다면 99% 발생했다고 해석
            1. `key != NULL`
            2. `Extra != Using index`
            3. `조회에 필요한 모든 컬럼이 인덱스에 포함되지 않은 경우`
```sql
create table t1 (c1 int primary key, c2 int, c3 int, index idx1(c2));
explain select * from t1 where c2 = 30;
```
```sql
type            : ref -- Non Unique Index + `=` 조건 (Index Lookup 수행)
possible_keys   : idx1
key             : idx1 -- 인덱스 사용
key_len         : 5
Extra           : NULL -- where 조건은 인덱스 접근 단계에서 완전히 처리됨
```
* `Extra=NULL` 은 back lookup이 발생했다는 의미가 아님
    * 표시할 추가 정보가 없다는 의미일 뿐
    * back lookup 여부와 직접적인 관계 없음
* 참고) `Extra=Using where` 역시 back lookup이 발생했다는 의미가 아님
## Covering Index
```sql
create table t1 (c1 int primary key, c2 int, c3 int, index idx1(c2));
explain select c1, c2 from t1 where c2 = 30;
```
```sql
type            : ref
possible_keys   : idx1
key             : idx1 
key_len         : 5
Extra           : Using index -- covering index 사용
```
* secondary index인 `idx1`에는 pk 컬럼이 자동으로 포함됨 (InnoDB 기준)
    * `c1`은 pk 컬럼이며, 그 값 자체가 row를 식별하는 pk 값
    * `idx1`은 개념적으로 `(c2, c1)` 구조를 가짐
* 조회에 필요한 모든 컬럼이 인덱스에 존재하므로 테이블 접근 불필요 → `covering index` 사용
<br>

# 고급 인덱스 스캔
## Index Skip Scan
### 크기가 큰 테이블 생성
* 확실하게 `skip scan`을 확인하기 위해 c1에는 1개의 값만 넣음
```sql
set session cte_max_recursion_depth = 500000; -- 재귀의 최대 반복 횟수 설정

create table t1 (c1 int, c2 int, index idx1(c1, c2));
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select 1, n from seq; -- 500000건의 데이터 삽입
analyze table t1; -- 통계 정보 갱신
```
### 컬럼 Cardinality 확인
* MySQL 옵티마이저는 컬럼의 `Cardinality`를 기반으로 조건 `Selectivity`를 추정
    * `Selectivity`는 컬럼 단위가 아닌 **조건 단위** 개념이므로 통계 정보로 직접 제공되지 않음
```sql
show index from t1;
```
```sql
Table       : t1
Column_name : c1
Cardinality : 1
-----------------------
Table       : t1
Column_name : c2
Cardinality : 499206
```
### 실행 계획 확인
```sql
explain select * from t1 where c2 = 1;
```
```sql
type            : range -- skip scan은 내부적으로 c1의 각 distict 값에 대해 c2 탐색을 반복
possible_keys   : idx1
key             : idx1
key_len         : 10 -- (c1(4) + null flag(1)) + (c2(4) + null flag(1))
rows            : 49920 -- skip scan으로 인해 약 10% 수준으로 감소 (추정값)
filtered        : 100.00
Extra           : Using where; Using index from skip scan -- Index Skip Scan 사용
```
### +) where절이 범위 조건일 경우 Skip Scan으로 풀릴까?
* `skip scan`의 선택 조건은 동등 조건이냐 범위 조건이냐가 아님
* 범위 조건일 경우, index 탐색 범위가 넓어져 조건 selectivity가 낮아지므로 skip scan이 선택되지 않을 가능성이 높아지는 것
* 결국 판단은 옵티마이저의 몫
## Loose Index Scan
* `Extra=Using index for group-by`
    * group by를 위해 추가적인 정렬이나 임시 테이블이 필요 없음을 의미
        * loose index scan
        * 일반 index scan + grouping
    * `explain analyze`나 `optimizer trace`로 정확한 실행 방식 확인 가능
### table 생성
```sql
set session cte_max_recursion_depth = 500000;

create table t1 (c1 int, c2 int, index idx1(c1, c2));
insert into t1 with recursive seq(n) as (select 1 union all select n+1 from seq where n < 1000), c1_list(c1) as (select 1 union all select c1+1 from c1_list where c1<1000) select c1, c1*10+n from seq join c1_list; -- `c1=1~1000`, `c2=1~1000` 의 데이터 삽입
analyze table t1;
```
### 실행계획 확인
```sql
explain select c1, min(c2) from t1 group by c1;
explain analyze select c1, min(c2) from t1 group by c1;
```
```sql
type            : range -- range 조건이 없지만
                        -- 인덱스를 그룹 단위로 건너뛰며 탐색하므로 range로 표시
possible_keys   : idx1
key             : idx1
key_len         : 10 -- (c1(4) + null flag(1)) + (c2(4) + null flag(1))
rows            : 1002 -- rows가 그룹수와 비슷하게 출력
filtered        : 100.00
Extra           : Using index for group-by -- 인덱스를 이용하여 group by 수행
                                           -- loose index scan으로 풀렸을 가능성이 있음
```
```sql
| -> Covering index skip scan for grouping on t1 using idx1  (cost=601 rows=1002) (actual time=0.112..8.04 rows=1000 loops=1)
 |
```
* `skip scan for grouping on t1 using idx1`
    * 인덱스를 사용하여 다음 그룹으로 건너뛰며(skip) 탐색
    * loose index scan의 증거
### optimizer trace 확인
```sql
set optimizer_trace = 'enabled=on';
select c1, min(c2) from t1 group by c1;
select * from information_schema.optimizer_trace;
```
```sql
-- Oprimizer Trace 결과 중 필요한 부분만 발췌

"group_index_range": { -- group by를 인덱스로 처리할 수 있는지 검사
                       -- loose index scan일 경우에만 나타남
    "potential_group_range_indexes": [ -- 후보 인덱스 목록
        {
            "index": "idx1", -- 후보1
            "covering": true, -- covering index 여부
            "rows": 1002,
            "cost": 501
        }
    ]
},

"best_group_range_summary": { -- 후보 중 실제로 선택된 전략
    "type": "index_group", -- loose index scan (group by를 인덱스로 처리)
    "index": "idx1", -- 사용된 인덱스
    "group_attribute": "c2", -- min/max 대상 컬럼
    "min_aggregate": true, -- min(c2)
    "max_aggregate": false,
    "distinct_aggregate": false,
    "rows": 1002,
    "cost": 501,
    "key_parts_used_for_access": [ -- 인덱스 탐색에 사용된 컬럼 목록
        "c1", -- 각 c1 그룹의 첫 row을 읽고
        "c2" -- min(c2) 계산
    ],
    "ranges": [], -- range 조건 없음
    "chosen": true -- loose index scan이 최종 선택됨
},
```