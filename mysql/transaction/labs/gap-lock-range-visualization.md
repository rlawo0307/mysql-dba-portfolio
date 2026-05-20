# 목차
* Gap Lock 범위 시각화
    * 단일 조건에서 gap lock 범위 확인
    * range 조건에서 gap lock 범위 확인
    * 오른쪽 끝 범위 gap lock 확인
    * 오른쪽 끝 범위 gap lock 확인 - 조건을 만족하는 row가 없는 경우
    * 왼쪽 끝 범위 gap lock 확인
    * 왼쪽 끝 범위 gap lock 확인 - 조건을 만족하는 row가 없는 경우
* 참고 자료
    * [→ lock에 대한 내용 자세히 보기](../locking.md)
<br><br>

# Gap Lock 범위 시각화
* [Index 종류 별 lock 범위 확인](locking-index-behavior.md) 실습에서 non-unique index에서 next-key lock과 gap lock이 발생하는지 확인했다
* 현재 실습에서는 gap lock으로 인해 실제 어떤 index 구간이 잠기는지 확인해보자
## 단일 조건에서 gap lock 범위 확인
### session 1
```sql
set session transaction isolation level repeatable read; -- InnoDB에서 gap lock은 기본적으로 repeatable read에서 phantom read 방지용으로 사용
set @@autocommit = off;

create table t1 (c1 int, index idx(c1));
insert into t1 values (0), (10), (20), (30), (40), (50);

start transaction;
select * from t1 where c1 = 30 for update;
```
### session 2
```sql
insert into t1 values (10); -- 성공
insert into t1 values (15); -- 성공
insert into t1 values (20); -- blocked
insert into t1 values (30); -- blocked
insert into t1 values (35); -- blocked
insert into t1 values (40); -- 성공
```
```text
0         10        20        30        40        50
|---------|---------(/////////]---------|---------| → 기본 next-key lock
|---------|---------|---------(/////////)---------| → 추가 보호 범위
|---------|---------(///////////////////)---------| → 최종 잠금 범위
```
* 추가 보호 범위
    * `idx`는 non-unique index이므로 `c1=30`이 하나인지 여러 개인지 보장할 수 없음
    * 따라서 InnoDB는 `30` 하나가 아니라 동일 key 범위를 기준으로 검색함
    * 이 과정에서 `c1=30` key 범위의 끝을 확인하기 위해 다음 distinct key 경계까지 scan하며 gap을 보호
* `insert 20`이 blocked되는 이유
    * 새로운 `c1=20` index key는 기존 20과 30 사이의 index 위치에 삽입됨
    * 이 `(20, 30)` 범위는 이미 gap lock 보호 범위에 포함됨
    * 즉, `20` record 자체에 lock이 걸린 것이 아니라, 새 `20`이 들어갈 인덱스 위치가 보호된 gap 안이므로 blocked됨
## range 조건에서 gap lock 범위 확인
### session 1
```sql
set session transaction isolation level repeatable read;
set @@autocommit = off;

create table t1 (c1 int, index idx(c1));
insert into t1 values (0), (10), (20), (30), (40), (50);

start transaction;
select * from t1 where c1 between 20 and 30 for update;
```
### session 2
```sql
insert into t1 values (5);  -- 성공
insert into t1 values (10); -- blocked
insert into t1 values (20); -- blocked
insert into t1 values (30); -- blocked
insert into t1 values (35); -- blocked
insert into t1 values (40); -- 성공
```
```text
0         10        20        30        40        50
|---------(///////////////////]---------|---------| → 기본 next-key lock
|---------|---------|---------(/////////)---------| → 추가 보호 범위
|---------(/////////////////////////////)---------| → 최종 잠금 범위
```
* 단일 조건과 마찬가지로, range 조건도 실제 lock 범위는 where 조건 자체가 아니라 index scan 범위를 기준으로 결정됨
* range 종료 지점을 확인하기 위해 next distinct key 경계까지 gap으로 보호
## 오른쪽 끝 범위 gap lock 확인
### session 1
```sql
set session transaction isolation level repeatable read;
set @@autocommit = off;

create table t1 (c1 int, index idx(c1));
insert into t1 values (0), (10), (20), (30), (40), (50);

start transaction;
select * from t1 where c1 >= 50 for update;
```
### session 2
```sql
insert into t1 values (35); -- 성공
insert into t1 values (40); -- blocked
insert into t1 values (50); -- blocked
insert into t1 values (60); -- blocked
```
```text
                                                           (∞)
0         10        20        30        40        50     supremum
|---------|---------|---------|---------(/////////]---------| → 기본 next-key lock
|---------|---------|---------|---------|---------(/////////) → 추가 보호 범위
|---------|---------|---------|---------(///////////////////) → 최종 잠금 범위
```
* 검색 범위가 마지막 index key 이후의 빈 구간을 가리키므로 `(50, ∞)` 구간이 gap으로 보호됨
## 오른쪽 끝 범위 gap lock 확인 - 조건을 만족하는 row가 없는 경우
### session 1
```sql
set session transaction isolation level repeatable read;
set @@autocommit = off;

create table t1 (c1 int, index idx(c1));
insert into t1 values (0), (10), (20), (30), (40), (50);

start transaction;
select * from t1 where c1 > 50 for update;
```
### session 2
```sql
insert into t1 values (35); -- 성공
insert into t1 values (40); -- 성공
insert into t1 values (45); -- 성공
insert into t1 values (50); -- blocked
insert into t1 values (60); -- blocked
```
```text
                                                           (∞)
0         10        20        30        40        50     supremum
|---------|---------|---------|---------|---------(/////////) → gap lock
|---------|---------|---------|---------|---------|---------| → 추가 보호 범위 없음
|---------|---------|---------|---------|---------(/////////) → 최종 잠금 범위
```
* 조건을 만족하는 row가 없기 때문에 record lock은 발생하지 않음
* 다음 distinct key가 조건을 만족하는지 확인할 필요 없으므로 추가 보호 범위도 발생하지 않음
* 검색 범위가 마지막 index key 이후의 빈 구간을 가리키므로 `(50, ∞)` 구간이 gap으로 보호됨
## 왼쪽 끝 범위 gap lock 확인
### session 1
```sql
set session transaction isolation level repeatable read;
set @@autocommit = off;

create table t1 (c1 int, index idx(c1));
insert into t1 values (0), (10), (20), (30), (40), (50);

start transaction;
select * from t1 where c1 <= 0 for update;
```
### session 2
```sql
insert into t1 values (-5); -- blocked
insert into t1 values (0); -- blocked
insert into t1 values (5); -- blocked
insert into t1 values (10); -- 성공
```
```text
(-∞)
infimum   0         10        20        30        40        50
(/////////]---------|---------|---------|---------|---------| → 기본 next-key lock
|---------(/////////)---------|---------|---------|---------| → 추가 보호 범위
(///////////////////)---------|---------|---------|---------| → 최종 잠금 범위
```
* 검색 범위가 첫 번째 index key 이전의 빈 구간을 가리키므로 `(-∞, 0)` 구간이 gap으로 보호됨
## 왼쪽 끝 범위 gap lock 확인 - 조건을 만족하는 row가 없는 경우
### session 1
```sql
set session transaction isolation level repeatable read;
set @@autocommit = off;

create table t1 (c1 int, index idx(c1));
insert into t1 values (0), (10), (20), (30), (40), (50);

start transaction;
select * from t1 where c1 < 0 for update;
```
### session 2
```sql
insert into t1 values (-5); -- blocked
insert into t1 values (0); -- 성공
insert into t1 values (5); -- 성공
insert into t1 values (10); -- 성공
```
```text
(-∞)
infimum   0         10        20        30        40        50
(/////////)---------|---------|---------|---------|---------| → gap lock
|---------|---------|---------|---------|---------|---------| → 추가 보호 범위 없음
(/////////)---------|---------|---------|---------|---------| → 최종 잠금 범위
```
* 조건을 만족하는 row가 없기 때문에 record lock은 발생하지 않음
* 다음 distinct key가 조건을 만족하는지 확인할 필요 없으므로 추가 보호 범위도 발생하지 않음
* 검색 범위가 첫 번째 index key 이전의 빈 구간을 가리키므로 `(-∞, 0)` 구간이 gap으로 보호됨