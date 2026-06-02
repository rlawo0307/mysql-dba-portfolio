# 목차
* Slow Query Log 활성화
* Slow Query Log 분석
* 참고 자료
    * [logging](../logging.md)
<br>

# Slow Query Log 결과 분석
* slow query log를 활성화하고 결과를 분석해보자
* slow query log의 역할을 이해해보자
## slow query log 활성화
```sql
set global slow_query_log = on;                 -- slow query log 활성화 (global variable)
set session long_query_time = 0.1;              -- 0.1초 이상 걸린 쿼리를 기록
set global log_queries_not_using_indexes = on;  -- 인덱스를 사용하지 않는 쿼리를 기록 (global variable)
                                                -- 데이터가 증가할수록 느려질 가능성이 있는 쿼리를 찾기 위한 목적
show variables like 'slow_query_log%';
show variables like 'long_query_time';
show variables like 'log_queries_not_using_indexes';
```
```sql
slow_query_log                  : ON
slow_query_log_file             : /var/lib/mysql/mysql-db-slow.log
long_query_time                 : 0.100000
log_queries_not_using_indexes   : ON
```
## Slow Query Log 분석
```sql
set session cte_max_recursion_depth = 500000;

create table t1(c1 int primary key auto_increment, c2 int, c3 varchar(100));
insert into t1(c2, c3) with recursive seq as
(
    select 1 as n
    union all
    select n+1 from seq where n < 500000
)
select n % 10, concat('row_', n) from seq;

select * from t1 where c2 = 1 order by c3;
```
### Slow Query Log 확인
```bash
sudo tail -n 100 /var/lib/mysql/mysql-db-slow.log
```
```bash
# Query_time: 0.160773  Lock_time: 0.000003 Rows_sent: 50000  Rows_examined: 550000
SET timestamp=1780371316;
select * from t1 where c2 = 1 order by c3;
```
* `Query_time`
    * 쿼리 전체 실행 시간
* `Lock_time`
    * lock을 기다린 시간
* `Rows_sent`
    * 클라이언트에게 반환한 row 수
* `Rows_examined`
    * MySQL이 실행 과정에서 확인한 row 수
    * 필터링, 정렬(filesort), 임시 처리 등의 과정에서 접근한 row가 포함될 수 있음
## 결과
* Query_time : `0.160773`
* Lock_time : `0.000003`
* Rows_sent : `50000`
* Rows_examined : `550000`
## 결론
실험 결과, 쿼리 전체 실행 시간은 `0.160773`초로 크게 느린 수준은 아니었다. 하지만 `50,000`건의 결과를 반환하기 위해 `550,000`건을 검사한 것을 확인할 수 있었다. 이는 현재 데이터 규모에서는 큰 문제가 되지 않을 수 있지만, 데이터가 증가할 경우 불필요하게 많은 row를 읽게 되어 성능 저하로 이어질 수 있다.

slow query log는 쿼리 실행 정보를 제공하여 성능 문제를 분석할 수 있도록 돕는 도구이다. 하지만 성능 저하 원인을 직접적으로 제공하지는 않으며, 원인 파악 및 분석은 DBA의 역할이다. DBA는 slow query log 결과를 통해 성능 문제가 의심되는 쿼리를 식별하고, 이후 explain 또는 explain analyze를 이용해 실행 계획을 분석함으로써 실제 원인을 파악해야 한다.
<br>