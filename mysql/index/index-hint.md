# 인덱스 HINT
* 옵티마이저의 인덱스 선택 판단에 유저가 개입하는 수단
* 종류
    * 유도 힌트
        * `USE INDEX`
        * `IGNORE INDEX`
    * 강제 힌트
        * `FORCE INDEX`
    * 병합 전략 제어 힌트
        * `INDEX_MERGE`
        * `NO_INDEX_MERGE`
## USE INDEX
* 옵티마이저가 고려할 인덱스 후보를 제한하는 힌트
* `possible_keys`를 지정하는 힌트
```sql
select * from t1 USE INDEX(idx1) where c1 = 10;
select /*+ index(t1, idx1) */ * from t1 where c1 = 10; 
```
## IGNORE INDEX
* 옵티마이저가 특정 인덱스를 사용하지 못하도록 무시시키는 힌트
* 다른 인덱스 또는 table scan을 선택하도록 유도
```sql
select * from t1 IGNORE INDEX(idx1) where c1 = 10;
select /*+ no_index(t1, idx1)*/ * from t1 where c1 = 10;
```
## FORCE INDEX
* 옵티마이저가 지정한 인덱스를 사용하도록 강제하는 힌트
* 잘못 사용하면 성능이 저하될 수 있음
```sql
select * from t1 FORCE INDEX(idx1) where c1 = 10;
select /*+ force_index(t1, idx1)*/ * from t1 where c1 = 10;
```
## INDEX_MERGE
* 옵티마이저가 여러 인덱스를 병합해서 조건을 만족하는 row를 찾도록 유도하는 힌트
```sql
create table t1 (c1 int, c2 int, index idx1(c1), index idx2(c2));
select /*+ index_merge(t1)*/ * from t1 where c1 = 10 or c2 = 20;
```
```sql
type            : index_merge
possible_keys   : idx1, idx2
key             : idx1, idx2
key_len         : 10
Extra           : Using union(idx1, idx2); Using where
```
## NO_INDEX_MERGE
* 옵티마이저가 인덱스 병합을 고려하지 못하도록 억제하는 힌트
```sql
create table t1 (c1 int, c2 int, index idx1(c1), index idx2(c2));
select /*+ no_index_merge(t1)*/ * from t1 where c1 = 10 or c2 = 20;
```
```sql
type            : ALL
possible_keys   : idx1, idx2
key             : NULL 
key_len         : NULL
Extra           : Using where
```