# INDEX란?
* 테이블의 데이터를 빠르게 찾기 위한 자료구조
* 실제 데이터는 테이블에 저장되고 인덱스가 그 위치를 가리키고 있음
## INDEX 구조
* `B-Tree`
    * Balanced Tree 자료구조
    * 모든 노드에 `키 + 데이터(또는 데이터 포인터)` 저장
    * 내부 노드에 실제 데이터가 존재함
    * leaf 노드끼리 연결되어 있지 않음
    * 탐색 시간 : `O(log N)`
    * 원하는 키를 찾으면 중간 노드에서 바로 종료 가능
    * 범위 검색에 비효율적
* `B+Tree`
    * Balanced Tree 자료구조
    * 내부 노드에 `키 + 포인터` 저장
    * **실제 데이터는 leaf 노드에 저장되어 있음**
    * leaf 노드끼리 좌우로 연결되어 있으며 정렬된 `linked list` 형태
    * 탐색 시간 : `O(log N)`
    * 범위 검색에 매우 유리
* InnoDB 인덱스의 기본 구조는 `B+Tree`
    * DB 특성 상 범위 검색이 많음
    * leaf 노드가 연결되어 있기 때문에 범위 검색 시 순차 접근 가능
    * 내부 노드에 데이터가 없어 더 많은 키를 저장할 수 있음
        * 한 노드가 더 많은 자식 노드를 가질 수 있음 (fan-out 증가)
        * 트리 높이 감소
        * 디스크 페이지 접근 횟수 최소화 (디스크 I/O 감소)
## INDEX 종류
<center>

|                     |종류|
|---------------------|---|
|구조 기준             |B+Tree Index<br>Hash Index|
|물리적 저장 구조 기준   |Clustered Index<br>Secondary Index|
|제약 / 기능 기준       |Primary Key Index<br>Unique Index<br>Non Unique Index|
|컬럼 구성 기준         |Single Column Index<br>Composite Index<br>Covering Index|
|컬럼 값 처리 기준      |Prefix Index<br>Functional Index (MySQL 8.0+)|
|InnoDB 내부 자동 인덱스|Adaptive Hash Index (AHI)|
</center>

* 구조 기준
    * `B+Tree Index`
        * B+Tree 기반 인덱스
        * DBMS에서 가장 널리 사용하는 인덱스 구조
        * 등치(=) 검색, 범위 검색, 정렬에 활용 가능
    * `Hash Index`
        * hash table 기반 인덱스
        * 키 값을 hash function으로 변환해 저장
        * 등치(=) 검색에 최적화
        * 범위 검색, 정렬, prefix 검색에 활용할 수 없음
        * InnoDB에서는 직접 생성 불가
        * InnoDB는 반복적으로 접근되는 B+Tree page에 대해 `Adaptive Hash Index(AHI)`를 내부적으로 생성하여 사용할 수 있음
* 물리적 저장 구조 기준
    * `Clustered Index`
        * 특징
            * 테이블의 실제 데이터(row)가 pk 기준으로 정렬된 구조로 저장되는 인덱스
            * InnoDB에서는 `primary key index = clustered index`
            * 테이블 당 하나만 존재 가능
            * Secondary Index의 기준점 역할
            * pk 검색 성능 매우 빠름
            * 범위 검색에 효율적
        * leaf node 구조
            * 모든 컬럼 데이터 (실제 row 데이터)
            * trx_id
            * roll_pointer[ → MVCC 구성요소](../transaction/transaction-mechanism.md)
        * clustered index 선택 우선순위
            1. primary key
            2. not null + unique 인덱스 중 첫 번째
            3. 내부적으로 생성한 row_id
    * `Secondary Index`
        * 특징
            * Clustered Index 가 아닌 모든 인덱스
            * 실제 데이터가 아니라 pk 값을 저장하고 있는 인덱스
            * 테이블 당 여러 개 생성 가능
            * pk가 길면 secondary index의 크기 증가
            * 전체 성능 저하를 유발할 수 있음
        * leaf node 구조
            * 인덱스 키 값
            * 해당 row의 pk 값
```text
            * 동작 방식 (`Bookmark Lookup`/ `Back Lookup`)
                1. Secondary Index 탐색
                2. leaf 노드에서 pk 값 획득
                3. Clustered Index로 재탐색
                4. 실제 row 데이터 접근
```
* 제약 / 기능 기준
    * `Primary Key Index`
        * pk로 생성되는 인덱스
        * InnoDB에서는 clustered index와 동일
        * 테이블의 실제 row 데이터가 저장되는 유일한 인덱스
    * `Unique Index`
        * unique 제약으로 생성되는 인덱스
        * 중복을 허용하지 않는 secondary index
    * `Non Unique Index`
        * 중복을 허용하는 secondary index
* 컬럼 구성 기준
    * `Single Column Index`
        * 하나의 컬럼을 기준으로 생성된 인덱스
        * 특정 컬럼에 대한 검색, 정렬, 범위 조건을 빠르게 처리 가능
    * `Composite Index`
        * 두 개 이상의 컬럼을 기준으로 생성된 인덱스
        * 다중 컬럼 검색에 효율적
        * 컬럼 순서 결정 전략
            * 선두 컬럼부터 조건에 사용되어야 인덱스 활용에 효과적 → `leftmost prefix rule`
            * `=` 조건이 걸린 컬럼을 앞에 두기
            * [selectivity](../optimization/optimizer-statistics.md)가 높은 컬럼을 앞에 두기
            * 범위 조건이 걸린 컬럼은 뒤에 두기
            * order by 컬럼을 인덱스에 포함
            * group by 컬럼을 인덱스 앞에 두기
            * 가능하다면 covering index 고려
    * `Covering Index`
        * 쿼리에 필요한 모든 컬럼이 인덱스에 포함된 상태
        * 인덱스에 저장된 컬럼 값으로 쿼리 결과를 구성할 수 있는 상태
        * data page(clustered index)에 접근하지 않아도 됨
        * clustered index lookup(pk lookup) 불필요
        * pk 컬럼은 따로 인덱스에 포함하지 않아도 covering index 조건을 만족할 수 있음
        * secondary index leaf node에는 항상 pk가 포함되어 있음
        * explain 실행 시 extra에 `Using index`가 표시됨
        * 장점
            * 디스크 I/O 감소
            * 쿼리 성능 향상
        * 단점
            * 컬럼 수가 증가할수록 index size 증가
            * buffer pool을 많이 차지하게 될 수 있음
            * write 비용 증가
            * 대부분의 경우 `select *` 쿼리는 covering index를 사용하기 어려움
        * 사용하는 경우
            * read-heavy 환경
            * 특정 조회 패턴이 반복되는 경우
            * 조회 컬럼이 제한적인 경우
* 컬럼 값 처리 기준
    * `Prefix Index`
        * 문자열 컬럼의 앞부분(접두사)만 인덱스로 만드는 방식
        * 문자열의 일부만 인덱싱해서 저장 공간 절약
        * varchar, text, blob 타입에 사용
            * 특히, text 또는 blob은 전체 컬럼에 대한 인덱싱을 허용하지 않음
        * 단, 인덱스 키 중복이 발생할 수 있음
            * 접두사 길이가 짧으면 다른 문자열이 같은 접두사로 인식될 수 있음
            * unique index 생성으로 중복 방지는 가능하지만 주의 필요!
    * `Functional Index` (MySQL 8.0+)
        * 컬럼 값에 함수를 적용한 결과를 인덱스로 생성
        * 컬럼에 함수가 적용된 조건에서도 인덱슬 탈 수 있게 해줌
            * 일반 인덱스는 컬럼에 함수가 적용되면 인덱스를 사용 할 수 없음
                * `lower(c1) = 'a'` → 인덱스 사용 불가
                * `c1 = lower('A')` → 인덱스 사용 가능
        * 인덱스 정의와 쿼리의 함수 표현식이 완전히 동일해야 사용 가능
        * 대소문자 무시 검색, 날짜 가공 검색에서 유용
            * `lower()`, `upper()`, `date()`, `year()` 등을 자주 사용할 때
* InnoDB 내부 자동 인덱스
    * `Adaptive Hash Index`
        * InnoDB가 내부적으로 자동 생성하는 인덱스
        * 자주 사용되는 B+Tree 페이지를 메모리 내부에 hash 구조로 캐싱
        * B+Tree 인덱스를 대체하지 않으며, 보조적인 캐시 역할 수행
        * **사용자가 직접 인덱스를 생성/수정/삭제할 수 없음**
        * equality(=) 검색에 매우 빠름
        * 범위 검색, 정렬, like 검색에는 사용되지 않음
## Index Scan 방식
* `Index Full Scan`
    * 인덱스 전체를 처음부터 끝까지 읽는 방식
    * 인덱스는 사용하지만 조건 필터링 효과가 거의 없음
    * 인덱스가 테이블보다는 훨씬 작기 대문에 조건 필터링이 없어도 읽는 양이 훨씬 작음
    * covering index일 경우 table full scan보다 압도적으로 빠름 
    * 정렬 수행 시, 인덱스는 이미 정렬된 상태이므로 추가적인 정렬(filesort)이 필요 없음
    * 인덱스는 자주 사용되므로 버퍼 풀에 존재할 가능성이 높기 때문에 디스크 I/O 없이 메모리에서 처리 가능
```text
* 동작 방식
    1. Secondary Index 탐색
    2. leaf 노드 전체 순차 스캔
    3. 모든 pk 획득
    4. Clustered Index 재탐색 (back lookup)
    5. 전체 row에 접근
```
* `Index Lookup`
    * 인덱스의 특정 키 값을 정확히 찾는 방식
    * 가장 빠른 인덱스 접근 방식
    * 접근하는 인덱스 페이지 수가 매우 적음
```text
* 동작 방식
    1. Secondary Index 탐색
    2. 조건 키로 B+Tree 탐색
    3. leaf 노드에서 pk 획득
    4. Clustered Index 재탐색 (back lookup)
    5. 1건의 row에 접근
```
* `Index Range Scan`
    * 특정 범위의 인덱스 키를 연속으로 읽는 방식
    * B+Tree 기반 index는 leaf 노드가 연결되어 있어 디스크 접근에 효율적
    * 결과 row 수가 많아질수록 성능 저하가 발생할 수 있음
    * back lookup 횟수가 증가할수록 성능 저하를 유발할 수 있음
```text
* 동작 방식
    1. Secondary Index 탐색
    2. 조건 시작 지점 탐색
    3. leaf 노드 범위 스캔
    4. 다수의 pk 획득
    5. Clustered Index 재탐색 (back lookup)
    6. 여러 건의 row에 접근
```
## Index Scan 최적화 전략
* `Index Skip Scan`
    * 복합 인덱스에서 먼저 나온 컬럼이 조건에 없어도 옵티마이저가 선두 컬럼의 가능한 값들을 건너뛰며 내부적으로 여러 번 index range scan을 수행
    * 옵티마이저가 인덱스 통계를 통해 선두 컬럼의 서로 다른 값 개수(distinct)를 파악
    * 선두 컬럼의 distinct 값이 적을 경우 사용
    * 후행 컬럼의 조건 selectivity가 높을 경우 사용
    * 테이블 크기가 크고 full scan이 부담되는 경우 사용
* `Loose Index Scan`
    * 인덱스에서 필요한 값만 loose하게 읽어 불필요한 row 접근을 건너뛰는 방식
    * 각 그룹마다 첫 번째 값만 필요한 경우 사용
        * `group by + min()/max()`, `distinct`, 집계 쿼리 등
        * 단, group by 컬럼이 인덱스 순서와 동일해야 하며 집계 대상 컬림이 다음 순서에 나와야 함
* `Backward Index Scan`
    * 인덱스를 정렬 반대 방향으로 탐색하는 방식
    * 마지막 leaf 노드부터 탐색하여 역순으로 읽어서 반환
        * B+Tree는 양방향으로 leaf 노드가 연결되어 있기 때문에 가능
    * 정렬 기준 컬럼에 인덱스가 있으며 내림차순으로 정렬할 경우 사용
    * order by 컬럼이 인덱스 순서와 다를 경우 사용 불가
    * 혼합 정렬일 경우 사용 불가
## Index Merge
* 하나의 인덱스로 처리할 수 없는 조건을 여러 개의 인덱스를 동시에 사용해서 해결하는 방식
* 옵티마이저가 index merge를 사용했다면 인덱스 설계가 부족하다는 신호!
* index merge는 옵티마이저의 임사방편이며, 가능하다면 복합 인덱스로 대체하는 것이 좋음
* 종류
    * `Index Merge Intersection`
        * 두 인덱스 결과의 교집합
        * ex) where a = 10 and b = 20;
    * `Index Merge Union`
        * 두 인덱스 결과의 합집합
        * 중복 제거 필요
        * ex) where a = 10 or b = 20;
    * `Index Merge Sort-Union`
        * range scan 결과를 정렬 후 merge
        * 비용이 커서 잘 안 쓰임
        * ex) where a < 10 or b < 20;
* 장점
    * 복합 인덱스가 없어도 다중 조건 처리 가능
    * 기존 단일 인덱스를 재활용 가능
* 단점
    * secondary index를 여러 개 스캔 해야 함
    * pk 목록을 병합하는데 비용 발생
    * 메모리 사용량 증가
    * 복합 인덱스보다 거의 항상 느림
```text
* 동작 방식
    1. idx1로 조건에 맞는 pk 목록 수집
    2. idx2로 조건에 맞는 pk 목록 수집
    3. 두 결과를 집합 연산
    4. 최종 pk 목록으로 row 접근
```
<br><br>

# Index의 장점과 한계
* 인덱스는 조회 성능을 크게 향상시키지만 항상 최선의 선택은 아님
* 옵티마이저는 full table scan과 index scan을 모두 후보로 두고 cost를 비교하여 실행 계획을 선택
* 경우에 따라 index scan보다 full table scan이 더 효율적일 수 있음
## 옵티마이저가 Index Scan을 선택하지 않는 경우
* 조건에 인덱스 컬럼이 사용되지 않은 경우
* 조건을 인덱스로 사용할 수 없는 형태로 작성한 경우 [→ 인덱스를 타지 않는 패턴](../index/index-antipatterns.md)
* 복합 인덱스 규칙을 위반한 경우
* index scan이 cost가 더 높은 경우
* 통계 정보가 부정확할 경우
* 옵티마이저 힌트 사용으로 스캔 방식을 강제한 경우
## Index의 비용 (Trade-Off)
* 저장 공간 비용 발생
    * 인덱스는 테이블 데이터와 별도로 저장됨
    * 인덱스가 많을수록 디스크에 공간이 더 많이 필요
* 쓰기 작업 비용 증가
    * insert, update, delete 시 인덱스도 함께 갱신되어야 함
    * 인덱스가 많을수록 쓰기 성능 저하
* 복잡성 증가
    * 불필요한 인덱스가 많으면 옵티마이저가 비효율적인 실행 계획을 선택할 수 있음
    * 인덱스가 많을수록 옵티마이저가 실행 계획을 결정하는데 더 많은 계산 필요
## Index 설계 시 고려사항
```text
1. 자주 조회하는 컬럼에만 인덱스 생성하기
2. 복합 인덱스는 필요한 컬럼만 최소한으로 포함하기
3. 조회 성능과 쓰기 성능의 균형 고려하기
4. 주기적인 인덱스 모니터링 및 튜닝 수행
```
<br><br>

# 인덱스와 정렬(ORDER BY)
* `Index-Driven Sort`
    * 인덱스를 사용한 정렬
    * 인덱스는 이미 정렬된 상태이므로 추가적인 작업이 필요하지 않음
    * 인덱스 순서대로 읽으면 자동으로 정렬된 상태
    * 발생 조건
        * order by 컬럼들이 인덱스에 포함되어 있어야 함
        * order by 순서와 인덱스 컬럼 순서가 일치해야 함
        * 복합 인덱스에서 선행 컬럼에 범위 조건이 사용될 경우, 후행 컬럼들의 정렬이 깨짐
            * 범위 조건 컬럼을 기준으로 여러 그룹으로 나뉨
            * 후속 컬럼들은 각 그룹 내에서만 정렬되어 있음
            * 전체 결과에 대해 정렬된 상태가 아니라서 추가 정렬 작업(`Using filesort`) 발생
    * 성능 상 유리한 경우
        * covering index인 경우
        * LIMIT이 있는 경우
* `Using filesort`
    * 인덱스를 활용하지 못해서 MySQL이 `별도의 정렬 작업` 수행
        * 메모리 또는 디스크 기반 정렬
        * 필요 시, 임시 테이블 사용
    * 대량 데이터일수록 비용이 많이 들고 느림
    * 발생 조건
        * order by 컬럼이 인덱스에 없는 경우
        * order by와 where 조건이 인덱스 순서를 깨뜨릴 경우
        * 복잡한 조합이나 함수가 포함된 order by