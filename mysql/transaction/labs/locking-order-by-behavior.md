# 목차
* filesort를 통한 정렬이 lock 획득에 미치는 영향
    * order by 단독 사용
    * where 조건 + order by
* index를 통한 정렬이 lock 획득에 미치는 영향
    * 인덱스가 정렬에만 사용되는 경우
    * 인덱스가 탐색 + 정렬에 사용되는 경우
* 참고 자료
    * [→ 인덱스 정렬 방식 (filesort / index driven sort)](../../index/index-basics.md)
    * [→ 인덱스 정렬 방식 (filesort / index driven sort) 비교 실습](../../index/labs/07-filesort-vs-index-ordering.md)
    * [→ 인덱스별 lock 범위 비교 실습](../labs/locking-index-behavior.md)
<br><br>

# filesort를 통한 정렬이 lock 획득에 미치는 영향
## 실험 준비(session 1)
```sql
create table t1(id int primary key, c1 int, c2 int, c3 int);
insert into t1 values (1, 1, 10, 100), (2, 1, 20, 200), (3, 1, 30, 300), (4, 2, 40, 400), (5, 2, 50, 500);

set autocommit = 0;
```
## order by 단독
### session 1
```sql
start transaction;
explain select * from t1 order by c2 for update; -- 실행 계획 확인
select * from t1 order by c2 for update; -- 실제 실행
```
```sql
type            : ALL -- full table scan
rows            : 5
filtered        : 100.00
Extra           : Using filesort
```
### session 2
```sql
update t1 set c3 = 999 where id = 1; -- blocked
update t1 set c3 = 999 where id = 2; -- blocked
update t1 set c3 = 999 where id = 3; -- blocked
update t1 set c3 = 999 where id = 4; -- blocked
update t1 set c3 = 999 where id = 5; -- blocked
```
## where 조건 + order by
### session 1
```sql
start transaction;
explain select * from t1 where c1 = 1 order by c2 for update;
select * from t1 where c1 = 1 order by c2 for update;
```
```sql
type            : ALL -- full table scan
rows            : 5
filtered        : 20.00
Extra           : Using where; Using filesort
```
### session 2
```sql
update t1 set c3 = 999 where id = 1; -- blocked
update t1 set c3 = 999 where id = 2; -- blocked
update t1 set c3 = 999 where id = 3; -- blocked
update t1 set c3 = 999 where id = 4; -- blocked
update t1 set c3 = 999 where id = 5; -- blocked
```
## 결과
* order by 단독 : filesort를 통해 정렬 수행, 스캔된 모든 row에 lock 획득
* order by + where : (동일)
## 결론
실험 결과, where 조건의 유무와 상관없이 인덱스가 없어 full table scan이 수행되는 경우, 스캔된 모든 row에 lock이 걸리는 것을 확인할 수 있다.

InnoDB는 조건을 먼저 평가한 뒤 lock을 거는 것이 아니라, **row를 읽는 시점에 lock을 획득**한 후 where 조건을 평가한다. 따라서 full table scan이 수행되는 경우, where 조건을 만족하지 않는 row까지 포함하여 스캔된 모든 row에 lock이 걸리게 된다. **정렬은 이후 filesort를 통해 수행되지만 lock 범위에는 직접적인 영향을 주지 않는다.**

즉, lock 범위는 where 조건이 아니라 실제 access path에 의해 결정된다.

인덱스가 존재하고 인덱스를 통해 정렬을 수행할 수 있는 경우에는 lock 범위가 어떻게 변하는지 알아보기 위해서는 추가적인 실험이 필요하다.
<br><br>

# index를 통한 정렬이 lock 획득에 미치는 영향
* 인덱스를 통해 정렬을 수행하더라도 where 조건이 없다면 index full scan 하게 됨
    * 즉, 정렬 방식과 관계없이 scan된 모든 row에 lock이 획득됨
* lock 범위 차이를 확인하기 위해서는 `where 조건`을 통해 scan 범위를 제한해야 함
## 인덱스를 통해 정렬 수행
* 정렬은 인덱스로 처리할 수 있지만 탐색에는 사용할 수 없는 경우
### session 1
```sql
create table t2(id int primary key, c1 int, c2 int, c3 int, c4 int, index idx1(c2, c3));
insert into t2 values (1, 1, 10, 100, 1000), (2, 1, 20, 200, 2000), (3, 1, 30, 300, 3000), (4, 2, 40, 400, 4000), (5, 2, 50, 500, 5000);

set autocommit = 0;
start transaction;
explain select * from t2 force index(idx1) where c1 = 1 order by c2, c3 for update;
select * from t2 force index(idx1) where c1 = 1 order by c2, c3 for update;
```
```sql
type            : index -- index full scan
possible_keys   : NULL -- 탐색에 사용될 수 있는 인덱스 없음
key             : idx1
key_len         : 10 -- 정렬에 c2, c3 사용. 인덱스 탐색에 c1을 사용하지 않음
rows            : 5
filtered        : 20.00
Extra           : Using where -- 정렬은 인덱스로 처리
                              -- where 조건을 인덱스 탐색에 사용하지 않음
                              -- where 조건은 나중에 필터링
```
### session 2
```sql
update t2 set c4 = 9999 where id = 1; -- blocked
update t2 set c4 = 9999 where id = 2; -- blocked
update t2 set c4 = 9999 where id = 3; -- blocked
update t2 set c4 = 9999 where id = 4; -- blocked
update t2 set c4 = 9999 where id = 5; -- blocked
```
## 인덱스를 통해 탐색 + 정렬 수행
### session 1
```sql
create table t3(id int primary key, c1 int, c2 int, c3 int, c4 int, index idx2(c1, c2, c3));
insert into t3 values (1, 1, 10, 100, 1000), (2, 1, 20, 200, 2000), (3, 1, 30, 300, 3000), (4, 2, 40, 400, 4000), (5, 2, 50, 500, 5000);

set autocommit = 0;
start transaction;
explain select * from t3 where c1 = 1 order by c2, c3 for update;
select * from t3 where c1 = 1 order by c2, c3 for update;
```
```sql
type            : ref -- index lookup
possible_keys   : idx2
key             : idx2
key_len         : 5 -- 인덱스 탐색에 c1 사용. c2, c3는 이미 정렬된 상태이므로 그대로 활용
rows            : 3
filtered        : 100.00
Extra           : NULL
```
### session 2
```sql
update t3 set c4 = 9999 where id = 1; -- blocked
update t3 set c4 = 9999 where id = 2; -- blocked
update t3 set c4 = 9999 where id = 3; -- blocked
update t3 set c4 = 9999 where id = 4; -- Query OK, 1 row affected (0.00 sec)
update t3 set c4 = 9999 where id = 5; -- Query OK, 1 row affected (0.00 sec)
```
## 결과
* 인덱스를 통해 정렬 수행 : 스캔된 모든 row에 lock 획득
* 인덱스를 통해 탐색 + 정렬 수행 : where 조건을 만족하는 row에만 lock 획득
## 결론
실험 결과, 인덱스를 통해 정렬을 수행할 수 있더라도 where 조건을 인덱스 탐색에 사용할 수 없다면 lock 범위가 좁혀지지 않는 것을 확인할 수 있다.

where 조건을 인덱스 탐색에 사용할 수 없는 경우, 현재 실험에서는 index full scan을 수행한다. 이 경우, 스캔된 모든 row에 lock이 걸리게 되며 where 조건은 이후 필터링 단계에서 적용된다. table full scan + filesort과 다른 점은, filesort는 lock 획득 이후 별도의 정렬이 수행되지만, 인덱스 정렬은 인덱스를 따라 row를 읽는 순서 자체에 포함된다. 그러나 두 정렬 방식 모두 lock 범위에 직접적인 영향을 주지 않는다. 

반면, where 조건과 정렬을 모두 처리할 수 있는 인덱스를 사용하는 경우, 필요한 범위만 탐색하면서 순서대로 row를 읽는다(자동으로 정렬 처리). 이 경우, 실제 scan 범위가 줄어들고 lock은 좁은 범위에 대해서만 획득된다.

즉, lock 범위는 단순히 인덱스를 통해 정렬을 수행할 수 있는지 여부가 아니라, where 조건을 포함한 실제 access path와 scan 범위에 의해 결정된다.