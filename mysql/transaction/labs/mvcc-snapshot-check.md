# 목차
* MVCC snapshot 확인
* 참고 자료
    * [→ 트랜잭션에 대한 내용 자세히 보기](../transaction-basics.md)
    * [→ MVCC에 대한 내용 자세히 보기](../transaction-mechanism.md)
<br><br>

# MVCC snapshot 확인
* MVCC snapshot이 트랜잭션 간 데이터 일관성과 독립성을 어떻게 보장하는지 확인해보자
    * 서로 다른 두 세션이 각각 독립적인 snapshot을 보고 있는지 확인
* `session 1`과 `session 2`는 같은 database의 다른 session으로 접속
### session 1
```bash
mysql -u user1 -h localhost -p db1
```
```sql
-- 실험 준비
select connection_id(); -- 12

create table t1(c1 int, c2 int);
insert into t1 values (1, 100), (2, 200), (3, 300);

-- 실험 시작
set autocommit = 0; -- autocommit 비활성화

start transaction; -- 트랜잭션1 시작
select * from t1; -- 이 시점에 snapshot 생성
                  -- read view는 트랜잭션이 처음 데이터에 접근할 때 생성됨
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
### session 2
```bash
mysql -u user1 -h localhost -p db1
```
```sql
select connection_id(); -- 13

start transaction; -- 트랜잭션2 시작
update t1 set c2 = 2000 where c1 = 2; -- 데이터 수정
commit; -- commit 수행

select * from t1;
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 | 2000 |     -- session2에서는 정상적으로 데이터가 변경됨
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
### session 1
```sql
select * from t1; -- 데이터가 바뀌었는지 확인
```
```sql
+------+------+
| c1   | c2   |
+------+------+
|    1 |  100 |
|    2 |  200 |     -- session1은 데이터가 그대로 유지
|    3 |  300 |
+------+------+
3 rows in set (0.00 sec)
```
## 결과
* session2에서 데이터를 변경한 후
    * session2에서는 변경된 데이터를 조회할 수 있음
    * session1에서는 데이터가 변경되지 않음
## 결론
session2에서 데이터를 변경하고 커밋하더라도, 이전에 시작된 session1의 트랜잭션에서는 데이터가 변경되지 않는다. 이는 각 트랜잭션은 트랜잭션이 시작될 때 생성된 MVCC snapshot을 기준으로 데이터를 조회하기 때문이다.

session1의 트랜잭션(`트랜잭션1`)은 session2의 트랜잭션(`트랜잭션2`)보다 먼저 시작되었다. 즉, 트랜잭션1은 시작 시점의 데이터를 그대로 참조하며 트랜잭션2의 변경 사항을 볼 수 없다. 이러한 MVCC 특징 덕분에 동시에 실행되는 트랜잭션 간에도 데이터 일관성과 독립성이 보장된다.
<br><br>