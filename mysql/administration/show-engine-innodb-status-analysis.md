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
## 섹션 별 예시 결과 및 해석 (정상 상태)
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
    * 값이 높다면, background thread의 redo flush/write activity가 존재하는 상태
### 활용 케이스
* flush가 밀리는 것 같을 때
* background maintenance 상태 확인이 필요할 때
* 내부 worker가 비정상적으로 바쁘거나 한가할 때
---
</details>
<details><summary>SEMAPHORES 섹션 해석</summary>

---
### SEMAPHORES
* InnoDB 내부 thread끼리 resource 경합이 발생하는지 보여주는 영역
    * mutex
    * read/write lock
    * spin wait
    * OS wait
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
* `reservation count`
    * thread가 wait queue에 들어간 횟수
    * resource를 바로 얻지 못해서 대기 상태에 진입한 횟수
* `signal count`
    * 대기 중이던 thread를 깨운 횟수
    * 기다리던 thread가 다시 실행 가능 상태로 전환된 횟수
    * 현재 reservation count(182)와 signal count(181)의 차이가 크지 않으므로 대기했던 thread 대부분이 정상적으로 처리된 것으로 해석 가능
* `RW-shared` : read lock 경합 상태
* `RW-excl` : write lock 경합 상태
* `RW-sx` : InnoDB 내부 synchronization 관련 경합 상태
* `spins`
    * CPU busy waiting 횟수
    * lock이 금방 풀릴 거라 예상하면 thread가 잠들지 않고 잠깐 CPU를 돌면서 기다림
    * 즉, 짧은 active wait
* `rounds`
    * spin loop 반복 횟수
    * 값이 크면 CPU를 많이 소모하면서 lock을 대기함을 의미
* `OS waits`
    * spin으로 해결이 안되서 OS scheduler에 의해 sleep한 횟수
    * 진짜 block
    * 값이 크면 contention이 발생하는 상태일 수 있음
* `Spin rounds per wait`
    * 실제 sleep에 들어가기 전, wait 1회 당 평균 spin 횟수
    * 현재 값이 0이므로 spin contention이 없는 상태
    * 값이 크면 CPU를 많이 소모하며 내부 contention이 발생하는 상태일 수 있음
### 활용 케이스
* 내부 contention이 의심될 때
* CPU 사용량은 많은데 SQL workload는 낮을 때
* mutex 병목이 의심될 때
* latch contention 분석이 필요할 때
---
</details>
<details><summary>(★) TRANSACTIONS 섹션 해석</summary>

---
### TRANSACTIONS
* InnoDB 내부 트랜잭션 / lock / purge 상태를 보여주는 영역
    * 현재 트랜잭션 상태
    * lock 보유 여부
    * lock wait 여부
    * purge 진행 상태
    * undo backlog
    * 오래 열린 트랜잭션
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
* `Trx id counter`
    * 현재 InnoDB transaction id counter
    * InnoDB가 트랜잭션을 생성할 때 사용하는 내부 ID
    * 새로운 트랜잭션 생성 시 증가
* `Purge done for trx's n:o < 78908`
    * purge가 어디까지 완료되었는지를 보여줌
        * purge = 더 이상 필요 없는 undo version cleanup
    * 현재 결과에서, 트랜잭션 78908 이전 undo 정리 완료
    * 현재 transaction id counter(78910)와 purge 완료 지점(78908)의 차이가 작으므로 purge backlog가 거의 없는 것으로 해석 가능
* `undo n:o < 0`
    * undo purge 처리 관련 내부 정보
    * 운영 관점에서는 보통 깊게 해석하지 않아도 됨
* `state: running but idle`
    * purge thread 상태
    * 현재 결과에서는, thread는 살아있지만 할 일이 없음
* `History list length 0`
    * purge 대기 중인 undo version backlog
    * 값이 0이면, undo backlog 없음. purge 정상
    * 값이 크면, purge가 밀려 undo version이 누적된 상태
        * 오래 열린 트랜잭션 때문에 purge가 진행되지 못하는 경우
        * update/delete 작업이 많아 undo 생성 속도가 purge 처리 속도보다 빠른 경우 등
* `LIST OF TRANSACTIONS FOR EACH SESSION:`
    * 현재 InnoDB 세션별 트랜잭션 상태
* `---TRANSACTION ..., not started`
    * 세션은 존재하지만 아직 트랜잭션이 시작되지 않은 상태
* `lock struct(s)` : lock 구조 개수
* `heap size` : lock metadata 메모리 사용량
* `row lock(s)` : row lock 개수
### 활용 케이스
* lock wait 분석
* blocked된 트랜잭션 찾기
* 오래 열린 트랜잭션 찾기
* purge backlog 확인
* undo 누적 상태 확인
* MVCC snapshot 문제 의심
---
</details>
<details><summary>(★) FILE I/O 섹션 해석</summary>

---
### FILE I/O
* InnoDB의 disk I/O thread 상태와 pending I/O 여부를 보여주는 영역
    * read thread
    * write thread
    * insert buffer thread
    * fsync 상태
    *  pending I/O 상태
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
* `I/O thread n state`
    * I/O thread의 현재 상태
        * `read thread` : page read 요청 처리
        * `write thread` : page write 요청 처리
        * `insert buffer thread` : change buffer 관련 I/O 처리
* 상태
    * `waiting for i/o request` : 처리할 I/O 작업 자체가 없는 상태
    * `waiting for completed aio requests` : I/O thread가 비동기 I/O 요청 완료를 기다리는 상태
    * `doing async i/o` : 비동기 I/O 작업 처리 중
    * `fsyncing` : disk flush 동기화 중
    * `waiting for flush` : flush 관련 작업 대기
    * 등
* `Pending normal aio reads`
    * 아직 완료되지 않은 비동기 read 요청 수
    * InnoDB는 read I/O thread를 여러 개 사용하며, I/O thread 별 queue 상태
        * 즉, 각 read thread 별 backlog 개수를 나타냄
    * 실제 출력 결과에서도 read thread가 1개가 아닌 여러개(4개)인 것을 확인할 수 있음
* `aio writes`
    * 아직 완료되지 않은 비동기 write 요청 수
    * write I/O thread 별 queue 상태
* `ibuf aio reads`
    * change buffer 전용 비동기 read queue 상태
    * 비어있으면 pending이 없는 것으로 해석 가능
* `Pending flushes (fsync)` : fsync 대기 상태
    * `log` : redo log flush 관련 fsync 대기
    * `buffer pool` : dirty page flush 관련 fsync 대기
    * 값이 크면 flush backlog가 쌓인 상태로 해석 가능
* `OS file reads`, `OS file writes`, `OS fsync`
    * MySQL/InnoDB 시작 이후 누적된 file I/O 횟수
    * 누적값이므로 값 자체보다는 증가 추세를 보는 것이 중요
* `reads/s`, `writes/s`, `fsyncs/s` : 최근 측정 구간 기준 초당 I/O 발생량
### 활용 케이스
* disk I/O 병목이 의심될 때
* pending read/write가 쌓이는지 확인할 때
* fsync 지연이 의심될 때
* buffer pool flush가 밀리는지 확인할 때
* redo log flush 지연이 의심될 때
---
</details>
<details><summary>INSERT BUFFER AND ADAPTIVE HASH INDEX 섹션 해석</summary>

---
### INSERT BUFFER AND ADAPTIVE HASH INDEX
* change buffer와 AHI 상태를 보여주는 영역
    * secondary index 변경 최적화 상태
    * change buffer merge 상태
    * AHI 사용 상태
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
* `Ibuf`
    * change buffer 상태
    * secondary index page가 buffer pool에 없을 때 변경 사항을 임시로 저장해 random I/O를 줄이는 최적화 기능
* `size` : 현재 change buffer 사용량
* `free list len` : change buffer에서 사용 가능한 free entry 수
* `seg size` : change buffer segment 전체 크기
* `merges` : change buffer에 쌓인 변경 사항을 실제 index page에 반영한 횟수
* `merged operations` : change buffer에서 실제 index page로 merge된 작업 수
* `Hash table size` : AHI 내부 hash table 크기
* `node heap has n buffer(s)` : AHI 관련 내부 메모리 사용 상태
* `hash searches/s` : AHI를 사용한 lookup 횟수
* `non-hash searches/s` : 일반 B+Tree lookup 횟수
### 활용 케이스
* secondary index write workload가 많을 때
* change buffer merge backlog가 의심될 때
* AHI 활용 여부 확인
* 내부 index optimization 상태 확인
---
</details>
<details><summary>(★) LOG 섹션 해석</summary>

---
### LOG
* InnoDB redo log / checkpoint / flush 상태를 보여주는 영역
    * redo log 기록 진행 상태
    * log flush 상태
    * dirty page flush 진행 상태
    * checkpoint 위치
    * redo pressure 확인
```sql
---
LOG
---
Log capacity                 104857600   -- 전체 redo log capacity
Log capacity used            104857600 
Log sequence number          12423264498 -- 현재 redo log의 최종 위치. redo 기록이 어디까지 진행됐는지를 의미
Log buffer assigned up to    12423264498 -- redo log buffer에 할당된 위치
Log buffer completed up to   12423264498 -- redo 처리 완료 기준 위치
Log written up to            12423264498 -- redo log file에 write 완료된 위치
Log flushed up to            12423264498 -- disk flush 완료된 위치
Added dirty pages up to      12423264498 -- dirty page 생성 기준 LSN
Pages flushed up to          12423264498 -- dirty page flush 완료 기준 LSN
Last checkpoint at           12423264498 -- 마지막 checkpoint 위치. recovery 기준점
Log minimum file id is       3793        -- 현재 redo log 최소 file id
Log maximum file id is       3793        -- 현재 redo log 최대 file id
59 log i/o's done, 0.00 log i/o's/second -- redo 관련 I/O 누적 횟수 및 최근 초당 redo I/O 발생량
```
* 해석 포인트
    * `Log sequence number` ≈ `Log written up to` ≈ `Log flushed up to`
        * redo 생성 / redo log file write / disk flush가 거의 같은 지점까지 진행된 상태
        * 즉, redo 처리가 밀리지 않고 정상적으로 따라가고 있는 상태
    * `Log sequence number`와 `Last checkpoint at`의 차이가 큰 경우
        * redo log가 생성된 최신 위치와 crash recovery 기준점([checkpoint](../availability/crash-recovery.md)) 사이 차이가 큰 경우
        * checkpoint lag가 큰 상태
        * redo log는 계속 생성되고 있지만 checkpoint가 그만큼 따라오지 못하는 상태
        * dirty page flush가 밀리거나 write workload가 높은 상황에서 발생할 수 있음
        * 차이가 계속 커지면 crash recovery 시, 재적용해야 할 redo 범위가 커질 수 있음
    * `Added dirty pages up to`와 `Pages flushed up to`의 차이가 큰 경우
        * dirty page가 발생한 최신 LSN과 dirty page flush가 완료된 LSN의 차이가 큰 경우
        * 메모리에는 변경된 dirty page가 많이 있는데, disk flush가 따라오지 못하는 상태
        * dirty page flush backlog가 큰 상태
        * buffer pool 내 modified page 증가 가능
        * checkpoint 진행도 함께 늦어질 수 있음
### 활용 케이스
* redo log flush 지연이 의심될 때
* checkpoint lag 확인
* dirty page flush backlog 확인
* write-heavy workload 상태 확인
* recovery 부담이 커질 수 있는 상태 확인
---
</details>
<details><summary>(★) BUFFER POOL AND MEMORY 섹션 해석</summary>

---
### BUFFER POOL AND MEMORY
* InnoDB buffer pool 및 메모리 사용 상태를 보여주는 영역
    * buffer pool 사용량
    * free page 상태
    * dirty page 상태
    * pending read/write 여부
    * page read/write activity
    * LRU 상태
```sql
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 0                      -- InnoDB 내부 메모리 할당 정보
Dictionary memory allocated 511416                  -- data dictionary 관련 메모리 사용량
Buffer pool size   8192                             -- buffer pool 전체 page 수
Free buffers       7017                             -- 아직 사용되지 않은 free page 수
Database pages     1171                             -- 실제 data page 또는 index page로 사용 중인 page 수
Old database pages 452                              -- LRU old sublist에 있는 page 수 
Modified db pages  0                                -- dirty page 수
Pending reads      0                                -- 아직 완료되지 않은 read 요청 수
Pending writes: LRU 0, flush list 0, single page 0  -- flush 대기 중인 write 요청 수
Pages made young 0, not young 0                     -- old page의 young 승격 통계
0.00 youngs/s, 0.00 non-youngs/s                    -- 최근 buffer pool page access의 LRU 처리 통계
Pages read 1027, created 144, written 269           -- 누적 page I/O 통계
0.00 reads/s, 0.00 creates/s, 0.00 writes/s         -- 최근 초당 page activity
No buffer pool page gets since the last printout    -- buffer pool access activity 요약 메시지
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s -- read-ahead 및 미사용 eviction 통계
LRU len: 1171, unzip_LRU len: 0                     -- LRU list 및 compressed page LRU 상태
I/O sum[0]:cur[0], unzip sum[0]:cur[0] 
```
* 해석 포인트
    * `Free buffers`가 매우 낮으면, buffer pool 여유 공간이 부족한 상태이므로 새 page 확보를 위해 eviction/flush 압박 발생 가능
    * `Modified db pages`가 크면, flush되지 않은 dirty page가 많이 쌓인 상태
    * `Pending writes`가 크면, flush가 밀리고 있는 상태
    * `evicted without access`가 높으면, 비효율적인 read-ahead 또는 cache 낭비 가능
---
</details>

## 섹션 별 예시 결과 및 해석 (deadlock 발생 시)
```sql
create table t1 (id int primary key, c1 int);
insert into t1 values (1, 100), (2, 200);

-- session 1
start transaction;
update t1 set c1 = c1 + 1 where id = 1;

-- session 2
start transaction;
update t1 set c1 = c1 + 1 where id = 2;

-- session 1
update t1 set c1 = c1 + 1 where id = 2; -- wait

-- session 2
update t1 set c1 = c1 + 1 where id = 1; -- ERROR 1213 (40001): Deadlock found when trying to get lock
```

<details><summary>전체 show engine innodb status 결과</summary>

```sql
*************************** 1. row ***************************
  Type: InnoDB
  Name: 
Status: 
=====================================
2026-05-12 08:10:43 138848807909056 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 40 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 5 srv_active, 0 srv_shutdown, 620 srv_idle
srv_master_thread log flush and writes: 0
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 385
OS WAIT ARRAY INFO: signal count 380
RW-shared spins 0, rounds 0, OS waits 0
RW-excl spins 0, rounds 0, OS waits 0
RW-sx spins 0, rounds 0, OS waits 0
Spin rounds per wait: 0.00 RW-shared, 0.00 RW-excl, 0.00 RW-sx
------------------------
LATEST DETECTED DEADLOCK
------------------------
2026-05-12 08:09:52 138849335457472
*** (1) TRANSACTION:
TRANSACTION 79961, ACTIVE 40 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1128, 2 row lock(s), undo log entries 1
MySQL thread id 14, OS thread handle 138848810022592, query id 20 localhost user1 updating
update t1
set c1 = c1 + 1
where id = 2

*** (1) HOLDS THE LOCK(S):
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79961 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 000000013859; asc     8Y;;
 2: len 7; hex 02000001251172; asc     % r;;
 3: len 4; hex 80000065; asc    e;;


*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79961 lock_mode X locks rec but not gap waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000002; asc     ;;
 1: len 6; hex 00000001385a; asc     8Z;;
 2: len 7; hex 01000000d612de; asc        ;;
 3: len 4; hex 800000c9; asc     ;;


*** (2) TRANSACTION:
TRANSACTION 79962, ACTIVE 18 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1128, 2 row lock(s), undo log entries 1
MySQL thread id 15, OS thread handle 138848808965824, query id 21 localhost user1 updating
update t1
set c1 = c1 + 1
where id = 1

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79962 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000002; asc     ;;
 1: len 6; hex 00000001385a; asc     8Z;;
 2: len 7; hex 01000000d612de; asc        ;;
 3: len 4; hex 800000c9; asc     ;;


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79962 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;;
 1: len 6; hex 000000013859; asc     8Y;;
 2: len 7; hex 02000001251172; asc     % r;;
 3: len 4; hex 80000065; asc    e;;

*** WE ROLL BACK TRANSACTION (2)
------------
TRANSACTIONS
------------
Trx id counter 79964
Purge done for trx's n:o < 79964 undo n:o < 0 state: running but idle
History list length 0
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 420324763603696, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763602888, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763601272, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763600464, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763599656, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763598848, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763597232, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763598040, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 420324763596424, not started
0 lock struct(s), heap size 1128, 0 row lock(s)
---TRANSACTION 79961, ACTIVE 91 sec
3 lock struct(s), heap size 1128, 2 row lock(s), undo log entries 2
MySQL thread id 14, OS thread handle 138848810022592, query id 20 localhost user1
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
1055 OS file reads, 632 OS file writes, 340 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.69 writes/s, 0.35 fsyncs/s
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
0.10 hash searches/s, 0.30 non-hash searches/s
---
LOG
---
Log capacity                 104857600
Log capacity used            104857600
Log sequence number          12423395509
Log buffer assigned up to    12423395509
Log buffer completed up to   12423395509
Log written up to            12423395509
Log flushed up to            12423395509
Added dirty pages up to      12423395509
Pages flushed up to          12423395509
Last checkpoint at           12423395509
Log minimum file id is       3793
Log maximum file id is       3793
107 log i/o's done, 0.05 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 0
Dictionary memory allocated 514112
Buffer pool size   8192
Free buffers       7011
Database pages     1177
Old database pages 454
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 1028, created 149, written 413
0.00 reads/s, 0.00 creates/s, 0.49 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 1177, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
0 read views open inside InnoDB
Process ID=1562, Main thread ID=138849095243456 , state=sleeping
Number of rows inserted 2, updated 3, deleted 0, read 3
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
Number of system rows inserted 25, updated 359, deleted 27, read 5112
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.45 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================

1 row in set (0.00 sec)
```
</details>
<details><summary>(★) LATEST DETECTED DEADLOCK 섹션 해석</summary>

---
### LATEST DETECTED DEADLOCK
* 가장 최근에 감지된 deadlock 정보를 보여주는 영역
    * deadlock에 참여한 트랜잭션
    * 각 트랜잭션이 보유한 lock
    * 기다리는 lock
    * rollback 대상 트랜잭션 등
* 정상 상태에서는 해당 섹션이 결과에 출력되지 않을 수 있음
```sql
------------------------
LATEST DETECTED DEADLOCK
------------------------
2026-05-12 08:09:52 138849335457472 -- deadlock이 감지된 시각과 OS thread handle
```
```sql
*** (1) TRANSACTION: -- deadlock에 참여한 트랜잭션 1
TRANSACTION 79961, ACTIVE 40 sec starting index read -- 트랜잭션 id 79961이 40초 동안 인덱스를 통한 row access 과정에서 lock을 기다리고 있음
mysql tables in use 1, locked 1 -- 1개 table을 사용중이고 lock과 관련된 상태임
LOCK WAIT 3 lock struct(s), heap size 1128, 2 row lock(s), undo log entries 1 -- LOCK WAIT : 현재 트랜잭션이 lock을 기다리는 상태
                                                                              -- 3 lock struct(s) : 트랜잭션이 가진 lock 구조 개수 = lock metadata 개수
                                                                              -- 2 row lock(s) : 트랜잭션과 관련된 row lock 개수(현재 1개는 보유중, 1개는 기다리는 중 = 총 2개)
                                                                              -- undo log entries 1 : 트랜잭션이 변경한 내용에 대한 undo log entry 수
MySQL thread id 14, OS thread handle 138848810022592, query id 20 localhost user1 updating
update t1
set c1 = c1 + 1
where id = 2
-- MySQL thread id : MySQL 세션 thread id. show processlist의 id와 연결해서 어떤 세션인지 확인 가능
-- user1@localhost 계정이 update를 수행 중에 lock wait 상태가 됨

*** (1) HOLDS THE LOCK(S): -- 트랜잭션이 현재 보유하고 있는 lock 정보
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79961 lock_mode X locks rec but not gap -- 어느 page의 어떤 종류의 lock인가를 설명하는 공통 metadata
-- space id : 이 record가 저장된 tablespace id
-- page no : tablespace 내부 page 번호
-- n bits : 해당 page의 record lock bitmap 크기
-- lock_mode X : exclusive lock. update를 위해 row를 배타적으로 잠근 상태
-- locks rec but not gap : gap lock이 아니라 record lock
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
-- heap no : page 내부에서 해당 record의 물리적 위치 번호. 사용자 입장에서는 row 식별 보조 정보로 보면 됨
-- PHYSICAL RECORD : page 안에 저장된 row record를 physical record 형태로 아래와 같이 보여줌
-- n_fields : physical record field 개수 (사용자 컬럼 + InnoDB 내부 metadata)
-- compact format : MySQL/InnoDB row format 종류 (redundant/compact/dynamic/compressed)
-- info bits : record의 internal flag. 현재 0은 특별한 상태 없음을 의미

 0: len 4; hex 80000001; asc     ;;             -- hex 80000001 = 1
                                                -- 즉, id(pk) = 1로 해석 가능

 1: len 6; hex 000000013859; asc     8Y;;       -- DB_TRX_ID (해당 row를 마지막으로 변경한 transaction id)
                                                -- 사용자 컬럼 아님

 2: len 7; hex 02000001251172; asc     % r;;    -- DB_ROLL_PTR (undo log 위치)
                                                -- 사용자 컬럼 아님(내부 관리용 컬럼)

 3: len 4; hex 80000065; asc    e;;             -- 두번째 사용자 컬럼 값
                                                -- hex 80000065 = 101
                                                -- 즉, c1 = 101로 해석 가능


*** (1) WAITING FOR THIS LOCK TO BE GRANTED: -- 트랜잭션이 획득하려고 기다리는 lock 정보
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79961 lock_mode X locks rec but not gap waiting -- 마지막에 waiting이 붙어있으면 아직 lock을 얻지 못한 상태
                                          -- 위의 보유하고 있는 lock 정보와 똑같아 보이지만 아래 physical record 부분이 다름
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000002; asc     ;; -- hex 80000002 = 2. 즉, id(pk) = 2인 row
 1: len 6; hex 00000001385a; asc     8Z;;
 2: len 7; hex 01000000d612de; asc        ;;
 3: len 4; hex 800000c9; asc     ;; -- hex 800000c9 = 201. 즉 c1 = 201
```
```sql
*** (2) TRANSACTION: -- deadlock에 참여한 트랜잭션 2
TRANSACTION 79962, ACTIVE 18 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1128, 2 row lock(s), undo log entries 1
MySQL thread id 15, OS thread handle 138848808965824, query id 21 localhost user1 updating
update t1
set c1 = c1 + 1
where id = 1

*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79962 lock_mode X locks rec but not gap
Record lock, heap no 3 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000002; asc     ;; -- id(pk) = 2 인 row에 대한 lock을 잡고 있음
 1: len 6; hex 00000001385a; asc     8Z;;
 2: len 7; hex 01000000d612de; asc        ;;
 3: len 4; hex 800000c9; asc     ;;


*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 480 page no 4 n bits 72 index PRIMARY of table `db1`.`t1` trx id 79962 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 4; hex 80000001; asc     ;; -- id(pk) = 1인 row에 대한 lock을 기다리는 중
 1: len 6; hex 000000013859; asc     8Y;;
 2: len 7; hex 02000001251172; asc     % r;;
 3: len 4; hex 80000065; asc    e;;

*** WE ROLL BACK TRANSACTION (2) -- deadlock을 해결하기 위해 트랜잭션 79962를 victim transaction으로 선택하여 rollback
```
### 활용 케이스
* deadlock 원인 분석
* 어떤 트랜잭션이 rollback 되었는지 확인
* lock 획득 순서 분석
---
</details>