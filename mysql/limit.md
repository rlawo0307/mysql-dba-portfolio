# LIMIT이란?
* 결과 집합에서 반환할 row 수를 제한하는 절
* 주로 페이징, 상위 `N`건 조회, 샘플 데이터 확인 용으로 사용
* 논리적 처리 순서
    * `from → where → order by → limit`
    * 정렬이 끝난 결과 집합에서 `limit`으로 잘라서 반환
* 물리적 처리
    * **옵티마이저는 limit을 이용해 조기 종료를 시도할 수 있음**
    * 조기 종료가 가능한 조건
        * `where + order by`가 인덱스 순서와 일치할 경우
        * 인덱스가 정렬을 보장할 경우
* **limit이 항상 적게 읽고 빠름을 보장하지 않음**
    * `offset`이 큰 경우
    * 정렬이 인덱스로 해결되지 않는 경우 등 (`filesort` 필요)
        * 단, `filesort`가 발생해도 필요한 상위 n개만 유지하므로 `filesort`를 단독으로 사용하는 것보다는 정렬 비용이 훨씬 작음
* **limit이 빠른 것이 아니다. limit이 전체 작업을 하지 않아도 되는 상황을 만드는 것이다**
<br>

# 기본 동작 방식
```sql
limit row_count
```
* 결과 집합 중 앞에서부터 `row_count`건만 반환
* 내부적으로는 조건과 정렬을 만족하는 row를 순서대로 읽다가 n 건이 채워지면 조기 종료
```sql
limit offset row_count
```
* 앞의 `offset`만큼은 버리고 그 다음 `row_count`건을 반환
* `offset`이 클수록 선형적으로 느려짐
<br>

# ORDER BY + LIMIT 실습
[→ order by + limit 실습 내용 자세히 보기](labs/explain-table-access-types.md)