# 목차
* Limit이 lock 범위에 미치는 영향
    * index가 없는 경우
    * index가 있는 경우
* 참고 자료
    * [→ limit에 대해 자세히 알아보기](../../optimization/limit.md)
    * [→ 인덱스별 lock 범위 비교 실습](../labs/locking-index-behavior.md)
<br><br>

# Limit이 lock 범위에 미치는 영향
* limit을 사용하면 상위 n건의 데이터만 조회한다. 이에 따라 lock 범위도 달라지는지 확인해보자
* session 2의 조회 조건
    * `id = 1` : session 1에서 lock을 걸었는지 확인용
    * `id = 2` : session 1에서 c1=1인 모든 row에 lock을 거는지 확인용
    * `id = 4` : session 1에서 c1=1이 아닌 row에 lock을 거는지 확인용 
## index가 없는 경우
### session 1
```sql
-- 실험 준비
create table t1(id int, c1 int, c2 int);
insert into t1 values (1, 1, 10), (2, 1, 20), (3, 1, 30), (4, 2, 40), (5, 2, 50);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화
start transaction; -- 트랜잭션 시작
select * from t1 where c1 = 1 limit 1 for update;
```
### session 2
```sql
update t1 set c2 = 99 where id = 1; -- blocked
update t1 set c2 = 99 where id = 2; -- blocked
update t1 set c2 = 99 where id = 4; -- blocked
```
## index가 있는 경우(1) - session2에서 full table scan할 경우
### session 1
```sql
-- 실험 준비
create table t2(id int, c1 int, c2 int, index idx1(c1));
insert into t2 values (1, 1, 10), (2, 1, 20), (3, 1, 30), (4, 2, 40), (5, 2, 50);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화
start transaction; -- 트랜잭션 시작
select * from t2 where c1 = 1 limit 1 for update;
```
### session 2
```sql
update t2 set c2 = 99 where id = 1; -- blocked
update t2 set c2 = 99 where id = 2; -- blocked
update t2 set c2 = 99 where id = 4; -- blocked
```
## index가 있는 경우(2) - session2에서 인덱스로 탐색할 경우
### session 1
```sql
-- 실험 준비
create table t3(id int primary key, c1 int, c2 int, index idx1(c1));
insert into t3 values (1, 1, 10), (2, 1, 20), (3, 1, 30), (4, 2, 40), (5, 2, 50);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화
start transaction; -- 트랜잭션 시작
select * from t3 where c1 = 1 limit 1 for update;
```
### session 2
```sql
update t3 set c2 = 99 where id = 1; -- blocked
update t3 set c2 = 99 where id = 2; -- Query OK, 1 row affected (0.01 sec)
update t3 set c2 = 99 where id = 4; -- Query OK, 1 row affected (0.01 sec)
```
## 결과
* 인덱스가 없는 경우
    * `id=1, 2, 4` 인 모든 row가 blocked됨
* 인덱스가 있는 경우 - session2에서 full table scan할 경우
    * `id=1, 2, 4` 인 모든 row가 blocked됨
* 인덱스가 있는 경우 - session2에서 index scan할 경우
    * session1이 실제로 선택한 row(`id=1`)만 blocked됨
    * 다른 row는 정상적으로 수정 가능
## 결론
실험 결과, limit을 사용하면 실제로 접근한 N건의 row 중심으로 lock 범위가 좁혀지는 것을 확인할 수 있다(`index가 있는 경우(2)`). 하지만 lock 범위가 단순히 limit 값만으로 결정되는 것은 아니라는 점에 주의해야 한다(`index가 없는 경우`,`index가 있는 경우(1)`). 특히, blocking 여부는 session1의 lock 범위뿐만 아니라 session2가 대상 row에 어떤 방식으로 접근하는지에 따라 다르게 나타났다.

테이블에 인덱스가 없는 경우(`index가 없는 경우`), session1은 조건(`c1=1`)을 만족하는 row를 찾기 위해 full table scan을 수행한다. 하지만 전체 row를 탐색한다고 해서 전체 row에 lock이 걸리는 것은 아니다. 실제로는 탐색 과정에서 조건을 만족한 row에 lock이 걸리게 된다. 특히 limit과 함께 사용될 경우, 조건을 만족하는 N건의 row를 찾으면 탐색이 즉시 종료되어 lock 범위를 좁힐 수 있다. 이런 상황에서 session2가 update 조건을 만족하는 row를 찾기 위해 full scan을 하게 될 경우, scan하는 도중에 lock이 걸린 row를 먼저 만난다면 lock을 대기하기 때문에 blocked되고, 이는 마치 update 대상 row가 잠긴 것처럼 보일 수 있다.

테이블에 인덱스가 존재하는 경우, session1에서 사용된 인덱스(`idx1`)는 non-unique index로, 조건(`c1=1`)을 만족하는 row를 탐색하는 과정에서 접근한 index entry를 기준으로 next-key lock을 적용한다. 이때, limit과 함께 쓰일 경우, 조건을 만족하는 N건의 row를 찾으면 탐색이 종료되므로 더 좁은 범위에 next-key lock이 적용된다.

이러한 상황에서 session2에 인덱스가 존재하지만 실제 update 수행에 인덱스가 사용되지 않는다면 인덱스가 없는 경우와 동일하게 동작한다(`index가 있는 경우(1)`). 반면 session2가 인덱스를 통해 update 대상 row에 직접 접근하는 경우(`index가 있는 경우(2)`), session1에서 lock을 적용한 row를 제외한 나머지 row에는 정상적으로 update가 가능하다.

인덱스는 session1에서는 lock 형태와 범위를 결정하며 limit은 정해진 lock 범위를 좀 더 좁힐 수 있다. session2에서는 인덱스를 통해 row에 접근하는 방식이 결정되며, 이는 blocking 여부에 직접적인 영향을 준다.

결론적으로, limit은 locking read 범위를 좁히는 데 영향을 줄 수 있지만, 실제 blocking 현상은 쿼리의 실행 계획과 접근 경로에 크게 의존한다. 따라서 blocking 여부만으로는 lock 범위를 단정할 수 없으며, 반드시 실행 계획과 함께 고려되어야 한다.