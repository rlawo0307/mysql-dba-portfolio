# 목차
* Buffer Pool Cache 동작 확인
    * cold cache 상태에서 조회
    * warm cache 상태에서 조회
* 참고 자료
    * [buffer pool 구조](../innodb-memory-architecture.md)
<br><br>

# Buffer Pool Cache 동작 확인
* InnoDB buffer pool이 data page와 index page를 메모리에 캐싱하는지 확인해보자
* cold cache와 warm cache 동작의 차이를 확인해보자
* buffer pool hit ratio를 계산하여 cache 효율과 I/O 발생 패턴을 분석해보자
## 실험 준비
```sql
set session cte_max_recursion_depth = 500000;

create table t1(id int primary key, c1 int, c2 int) engine = InnoDB;
insert into t1 with recursive seq as (select 1 as n union all select n+1 from seq where n < 500000) select n, n, n*100 from seq;
```
## Cold Cache 상태에서 조회
### buffer pool 상태 제어 옵션 끄기
* 기본적으로 MySQL 재시작 시, buffer pool이 비워짐(cold cache 상태)
* MySQL 8.0 이상에서는 성능 개선을 위해 cache를 저장하고 복구하는 옵션이 추가됨
    * `innodb_buffer_pool_dump_at_shutdown`
        * MySQL 종료 시 buffer pool에 어떤 page가 있었는지 기록(dump)할지 여부
        * 설정파일에서 수정하거나 MySQL에서 동적으로 수정 가능
    * `innodb_buffer_pool_load_at_startup`
        * MySQL 시작 시 이전에 저장된 buffer pool 상태를 다시 메모리에 로드할지 여부
        * 설정 파일(.cnf)에서 수정한 뒤 MySQL 재시작 필요
#### 설정 파일 수정
```ini
# /etc/mysql/mysql.conf.d/mysqld.cnf
[mysqld]
innodb_buffer_pool_dump_at_shutdown=OFF
innodb_buffer_pool_load_at_startup=OFF
```
#### MySQL 재시작
```bash
sudo systemctl restart mysql
sudo mysql -uroot -p
```
#### 설정 변수 확인
```sql
show variables like 'innodb_buffer_pool_%_at_%';
```
```sql
+-------------------------------------+-------+
| Variable_name                       | Value |
+-------------------------------------+-------+
| innodb_buffer_pool_dump_at_shutdown | OFF   |
| innodb_buffer_pool_load_at_startup  | OFF   |
+-------------------------------------+-------+
2 rows in set (0.00 sec)
```
### 쿼리 실행 전/후 buffer pool hit ratio 비교
* `Innodb_buffer_pool_read_requests`
    * InnoDB가 page를 읽으려고 시도한 총 횟수(logical read)
    * 전체 읽기 요청 수(memory + disk)
    * row 수가 아닌 page 접근 요청 수
* `Innodb_buffer_pool_reads`
    * buffer pool에 page가 없어서 disk에서 실제로 읽은 횟수(physical read)
    * 실제 I/O 발생 횟수
    * cache miss일 때 증가
* `hit ratio` = `1 - (cache miss 증가량 / 전체 읽기 요청 증가량)`
#### before
* MySQL 시작 시 수행되는 내부 작업이나 이전에 실행한 실습 쿼리로 인해 값이 0이 아닐 수 있음
```sql
flush status; -- 상태 변수 초기화 (buffer pool의 page는 제거하지 않음)
show global status where variable_name in ('Innodb_buffer_pool_read_requests', 'Innodb_buffer_pool_reads');
```
```sql
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| Innodb_buffer_pool_read_requests | 16199 |
| Innodb_buffer_pool_reads         | 1036  |
+----------------------------------+-------+
2 rows in set (0.00 sec)
```
#### after
```sql
select count(*) from t1 where c1 between 1 and 100000;

show global status where variable_name in ('Innodb_buffer_pool_read_requests', 'Innodb_buffer_pool_reads');
```
```sql
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| Innodb_buffer_pool_read_requests | 18308 |
| Innodb_buffer_pool_reads         | 1134  |
+----------------------------------+-------+
2 rows in set (0.00 sec)
```
## Warm Cache 상태에서 조회
### 동일한 쿼리 재조회 후 hit ratio 확인
```sql
select count(*) from t1 where c1 between 1 and 100000;

show global status where variable_name in ('Innodb_buffer_pool_read_requests', 'Innodb_buffer_pool_reads');
```
```sql
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| Innodb_buffer_pool_read_requests | 19797 |
| Innodb_buffer_pool_reads         | 1134  |
+----------------------------------+-------+
2 rows in set (0.00 sec)
```
## 결과
* cold cache
    * read_requests 증가량 : 18308-16199 = `2109`
    * reads 증가량 : 1134-1036 = `98`
    * hit ratio : 1 - (98 / 2019) = 0.9535 → `약 95.35%`
* warm cache
    * read_requests 증가량 : 19797-18308 = `1489`
    * reads 증가량 : 1134-1134 = `0`
    * hit ratio : 1 - (0 / 1489) = 1 → `100%`
## 결론
cold cache 상태에서는 전체 읽기 요청 수(Innodb_buffer_pool_read_requests)와 cache miss 횟수(Innodb_buffer_pool_reads)가 함께 증가하는 것을 확인할 수 있다. 이는 필요한 page 중 일부가 buffer pool에 존재하지 않아 disk에서 직접 읽혔음을 의미한다.

단, cold cache 상태에서도 hit ratio가 0%가 아닌 약 95%로 나타나는 것을 통해 모든 읽기 요청이 disk read로 이어지지 않음을 알 수 있다. 이는 InnoDB가 데이터를 row 단위가 아닌 page 단위로 읽기 때문이다. 한 번 disk에서 읽은 page는 buffer pool에 적재되며, 해당 page에 포함된 다른 row에 대한 접근은 buffer pool에서 처리되므로 추가적인 disk I/O가 발생하지 않는다.

반면, warm cache 상태에서는 전체 읽기 요청 수는 증가했지만 cache miss 수는 증가하지 않았다. 이는 동일한 조회에 필요한 page가 이미 buffer pool에 적재되어 있어 disk I/O 없이 buffer pool에서 처리되었음을 의미한다.

추가로, 같은 쿼리를 실행했지만 cold cache와 warm cache의 전체 읽기 요청 수 증가량이 다르게 나타나는 것을 확인할 수 있다. cold cache 상태에서는 필요한 page를 처음 찾는 과정에서 더 많은 page 접근이 발생할 수 있으며, read-ahead와 같은 내부 최적화 동작의 영향으로 접근 횟수가 증가할 수 있다. 반면, warm cache 상태에서는 이미 필요한 page가 buffer pool에 적재되어 있어 보다 직접적이고 효율적인 접근이 이루어지므로 불필요한 page 접근이 줄어들 수 있다.