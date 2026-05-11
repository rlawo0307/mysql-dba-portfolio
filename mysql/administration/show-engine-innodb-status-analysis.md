# SHOW ENGINE INNODB STATUS 란?
* InnoDB 스토리지 엔진의 내부 상태를 텍스트 형태로 출력하는 진단 명령어
* MySQL 서버 안에서 InnoDB가 현재 어떤 상태인지 한번에 보여주는 InnoDB 내부 상태 dump
* 특징
    * 실행 시점의 InnoDB 상태를 보여주는 snapshot 형태의 진단 도구
        * 과거 이력을 계속 저장해두는 로그가 아님
    * 결과가 테이블 형태로 정규화 되어 있지 않고, 사람이 읽을 수 있는 긴 텍스트 형태로 출력
        * 자동 분석에는 불편하지만, 한 번에 많은 내부 정보를 볼 수 있음
    * 단독으로 쓰기보다는 다른 모니터링 도구와 함께 사용하는 것을 권장
* 활용 케이스
    * lock wait 원인 분석
    * deadlock 분석
    * 오래 열린 트랜잭션 확인
    * undo / purge 상태 확인
    * buffer pool 상태 확인
    * redo log / checkpoint 상태 확인
    * file I/O 대기 확인
    * InnoDB 내부 경합 확인
## 출력 구조 overview
```text
BACKGROUND THREAD                     → background worker 상태
SEMAPHORES                            → 내부 mutex / rw-lock 경합
LATEST DETECTED DEADLOCK              → 최근 deadlock 정보
TRANSACTIONS                          → 트랜잭션 / lock 상태
FILE I/O                              → disk I/O 상태
INSERT BUFFER AND ADAPTIVE HASH INDEX → change buffer / AHI 상태
LOG                                   → redo log / checkpoint 상태
BUFFER POOL AND MEMORY                → buffer pool 상태
ROW OPERATIONS                        → row-level workload 상태
```
## 섹션 별 예시 결과 및 해석 
<details><summary>전체 show engine innodb status 결과</summary>

```sql
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
=====================================
2026-05-11 08:47:59 135459646662336 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 5 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 1 srv_active, 0 srv_shutdown, 228 srv_idle
srv_master_thread log flush and writes: 0
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 182
OS WAIT ARRAY INFO: signal count 181
RW-shared spins 0, rounds 0, OS waits 0
RW-excl spins 0, rounds 0, OS waits 0
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 0.00 RW-shared, 0.00 RW-excl, 0.00 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 78910
Purge done for trx's n:o < 78908 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 416934790794400, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790793592, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790792784, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790791976, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790791168, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790790360, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790789552, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790788744, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (read thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (write thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:
Pending flushes (fsync) log: 0; buffer pool: 0
1100 OS file reads, 389 OS file writes, 179 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 2162, seg size 2164, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 3 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log capacity                 104857600
Log capacity used            104857600
Log sequence number          12423264498
Log buffer assigned up to    12423264498
Log buffer completed up to   12423264498
Log written up to            12423264498
Log flushed up to            12423264498
Added dirty pages up to      12423264498
Pages flushed up to          12423264498
Last checkpoint at           12423264498
Log minimum file id is       3793
Log maximum file id is       3793
59 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 0
Dictionary memory allocated 511416
Buffer pool size   8192
Free buffers       7017
Database pages     1171
Old database pages 452
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 1027, created 144, written 269
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 1171, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1595, Main thread ID=135459107763904 , state=sleeping
Number of rows inserted 0, updated 0, deleted 0, read 0
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
Number of system rows inserted 17, updated 348, deleted 42, read 5109
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

1 row in set (0.00 sec)
```
</details>
<details><summary>BACKGROUND THREAD 섹션 해석</summary>

---
### BACKGROUND THREAD
* InnoDB 내부 background worker thread 상태를 보여주는 영역
    * foreground thread : 사용자 쿼리를 처리
    * background thread : 내부 유지보수 작업 담당
* dirty page flush, purge, change buffer merge, checkpoint 관련 작업 등을 수행
```sql
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 1 srv_active, 0 srv_shutdown, 228 srv_idle
srv_master_thread log flush and writes: 0
```
* `srv_master_thread loops` : InnoDB master thread의 loop 상태
    * `1 srv_active`
        * background thread가 실제 작업을 수행한 횟수
        * 현재 값이 매우 낮으므로 특별한 내부 작업이 거의 없는 상태로 해석 가능
    * `0 srv_shutdown`
        * shutdown 관련 작업 수행 횟수
        * 운영 중 정상 상태에서는 보통 0
        * MySQL 종료 과정에서 증가할 수 있음
    * `228 srv_idle`
        * 할 일이 없어 idle 상태였던 횟수
        * srv_active와 비교해보면 InnoDB background workload 수준을 대략적으로 판단할 수 있음
            * idle >> active : background maintenance workload가 거의 없는 한가로운 상태
            * idle ≈ active : 일정 수준의 background 작업이 지속적으로 발생하는 상태
            * active >> idle : 내부 작업이 활발한 상태
        * 단, 이 값만으로는 문제 여부를 단정할 수 없으며, 다른 섹션과 함께 해석하는 것이 좋음
* `srv_master_thread log flush and writes: 0`
    * background thread가 redo log flush/write를 수행한 횟수
    * 값이 0이면, 현재 특별한 flush 작업이 없는 상태
    * 값이 높다면, background flush 작업이 활발한 상태
### 활용 케이스
* flush가 밀리는 것 같을 때
* background maintenance 상태 확인이 필요할 때
* 내부 worker가 비정상적으로 바쁘거나 한가할 때
---
</details>
<details><summary>SEMAPHORES 섹션 해석</summary>

### SEMAPHORES
* InnoDB 내부 thread끼리 resource 경합이 발생하는지 보여주는 영역
```sql
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 182
OS WAIT ARRAY INFO: signal count 181
RW-shared spins 0, rounds 0, OS waits 0
RW-excl spins 0, rounds 0, OS waits 0
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 0.00 RW-shared, 0.00 RW-excl, 0.00 RW-sx
```
</details>
<details><summary>TRANSACTIONS 섹션 해석</summary>

```sql
------------
TRANSACTIONS
------------
Trx id counter 78910
Purge done for trx's n:o < 78908 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 416934790794400, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790793592, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790792784, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790791976, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790791168, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790790360, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790789552, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 416934790788744, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
```
</details>
<details><summary>FILE I/O 섹션 해석</summary>

```sql
--------
FILE I/O
--------
I/O thread 0 state: waiting for completed aio requests (insert buffer thread)
I/O thread 1 state: waiting for completed aio requests (read thread)
I/O thread 2 state: waiting for completed aio requests (read thread)
I/O thread 3 state: waiting for completed aio requests (read thread)
I/O thread 4 state: waiting for completed aio requests (read thread)
I/O thread 5 state: waiting for completed aio requests (write thread)
I/O thread 6 state: waiting for completed aio requests (write thread)
I/O thread 7 state: waiting for completed aio requests (write thread)
I/O thread 8 state: waiting for completed aio requests (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:
Pending flushes (fsync) log: 0; buffer pool: 0
1100 OS file reads, 389 OS file writes, 179 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
```
</details>
<details><summary>INSERT BUFFER AND ADAPTIVE HASH INDEX 섹션 해석</summary>

```sql
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 2162, seg size 2164, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 3 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
```
</details>
<details><summary>LOG 섹션 해석</summary>

```sql
---
LOG
---
Log capacity                 104857600
Log capacity used            104857600
Log sequence number          12423264498
Log buffer assigned up to    12423264498
Log buffer completed up to   12423264498
Log written up to            12423264498
Log flushed up to            12423264498
Added dirty pages up to      12423264498
Pages flushed up to          12423264498
Last checkpoint at           12423264498
Log minimum file id is       3793
Log maximum file id is       3793
59 log i/o's done, 0.00 log i/o's/second
```
</details>
<details><summary>BUFFER POOL AND MEMORY 섹션 해석</summary>

```sql
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 0
Dictionary memory allocated 511416
Buffer pool size   8192
Free buffers       7017
Database pages     1171
Old database pages 452
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 1027, created 144, written 269
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 1171, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
```
</details>
<details><summary>ROW OPERATIONS 섹션 해석</summary>

```sql
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1595, Main thread ID=135459107763904 , state=sleeping
Number of rows inserted 0, updated 0, deleted 0, read 0
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
Number of system rows inserted 17, updated 348, deleted 42, read 5109
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
```
