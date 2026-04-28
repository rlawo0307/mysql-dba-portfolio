# 목차
* Replication 동작 중 binlog 흐름 확인 실습
    * replication 구축(1 source + 1 replica)
    * 현재 source의 binlog 확인
        * 현재 사용중인 binlog 확인
        * 현재 source 서버에 존재하는 모든 binlog 파일 목록 확인
    * 트랜잭션 실행 후 binlog 확인
    * replica에서 수신 및 적용 확인
    * replica에서 relay log 확인
    * SQL thread 적용 확인
    * lag 실험
* 참고 자료
<br><br>

# Replication 동작 중 binlog 흐름 확인 실습
## 0. [replication 구축(1 source + 1 replica)](replication-setup-and-verification.md)
## 1. 현재 source의 binlog 확인
### 현재 사용중인 binlog 확인
```sql
show master status;
```
```sql
File                : mysql-bin.000016 -- 현재 쓰고 있는 binlog 파일
Position            : 529 -- binlog 파일 내부의 바이트 위치(offset)

-- 지금까지 실행된 트랜잭션들의 ID 집합
Executed_Gtid_Set   : 82141e86-396c-11f1-8939-000c29ecd7e8:1    -- 서버 A : 트랜잭션 1 실행됨 
                      aaa902cd-ecb6-11f0-bda3-000c2921ba33:1-65 -- 서버 B : 트랜잭션 1~65 실행됨
```
#### 서버 A, B 확인하기
<details>
<summary>방법 1) 서버 UUID 조회</summary>

```sql
select @@server_uuid;
```
* source
```sql
+--------------------------------------+
| @@server_uuid                        |
+--------------------------------------+
| aaa902cd-ecb6-11f0-bda3-000c2921ba33 | -- 서버 B의 UUID와 동일. 즉 서버 B = source 서버
+--------------------------------------+
1 row in set (0.00 sec)
```
* replica
```sql
+--------------------------------------+
| @@server_uuid                        |
+--------------------------------------+
| 82141e86-396c-11f1-8939-000c29ecd7e8 | -- 서버 A의 UUID와 동일. 즉 서버 A = replica 서버
+--------------------------------------+
1 row in set (0.00 sec)
```
</details>
<details>
<summary>방법 2) auto.cnf 파일 확인</summary>

```bash
sudo cat /var/lib/mysql/auto.cnf
```
#### source
```bash
[auto]
server-uuid=aaa902cd-ecb6-11f0-bda3-000c2921ba33 # 서버 B = source 서버
```
#### replica
```bash
[auto]
server-uuid=82141e86-396c-11f1-8939-000c29ecd7e8 # 서버 A = replica 서버
```
</details>

### 현재 source 서버에 존재하는 모든 binlog 파일 목록 확인
```sql
show binary logs;
```
```sql
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000001 |       180 | No        |
| mysql-bin.000002 |       180 | No        |
| mysql-bin.000003 |       180 | No        |
| mysql-bin.000004 |      1824 | No        |
| mysql-bin.000005 |       220 | No        |
| mysql-bin.000006 |      1433 | No        |
| mysql-bin.000007 |       197 | No        |
| mysql-bin.000008 |       816 | No        |
| mysql-bin.000009 |      2543 | No        |
| mysql-bin.000010 |       765 | No        |
| mysql-bin.000011 |       368 | No        |
| mysql-bin.000012 |      1950 | No        |
| mysql-bin.000013 |       236 | No        |
| mysql-bin.000014 |      1988 | No        |
| mysql-bin.000015 |       491 | No        |
| mysql-bin.000016 |       529 | No        | -- 현재 쓰고 있는 binlog
+------------------+-----------+-----------+
16 rows in set (0.01 sec)
```
## 2. 트랜잭션 실행 후 binlog 확인
```sql
insert into t1 values (5, 500);
show master status;
show binary logs;
```
```sql
File                : mysql-bin.000016 
Position            : 805 

-- 지금까지 실행된 트랜잭션들의 ID 집합
Executed_Gtid_Set   : 82141e86-396c-11f1-8939-000c29ecd7e8:1,
                      aaa902cd-ecb6-11f0-bda3-000c2921ba33:1-66 -- source 서버에서 실행된 트랜잭션이 증가
```
```sql
+------------------+-----------+-----------+
| Log_name         | File_size | Encrypted |
+------------------+-----------+-----------+
| mysql-bin.000001 |       180 | No        |
| mysql-bin.000002 |       180 | No        |
| mysql-bin.000003 |       180 | No        |
| mysql-bin.000004 |      1824 | No        |
| mysql-bin.000005 |       220 | No        |
| mysql-bin.000006 |      1433 | No        |
| mysql-bin.000007 |       197 | No        |
| mysql-bin.000008 |       816 | No        |
| mysql-bin.000009 |      2543 | No        |
| mysql-bin.000010 |       765 | No        |
| mysql-bin.000011 |       368 | No        |
| mysql-bin.000012 |      1950 | No        |
| mysql-bin.000013 |       236 | No        |
| mysql-bin.000014 |      1988 | No        |
| mysql-bin.000015 |       491 | No        |
| mysql-bin.000016 |       805 | No        | -- 현재 사용중인 binlog 파일의 offset 증가
+------------------+-----------+-----------+
16 rows in set (0.00 sec)
```
## 3. replica에서 수신 및 적용 확인
```sql
show replica status\G
```
```sql
Source_Host             : 192.168.111.129 -- source IP
Source_User             : binlog_reader -- 해당 계정으로 binlog를 가져옴
Source_Log_File         : mysql-bin.000016 -- 사용중인 binlog
Read_Source_Log_Pos     : 805 -- IO thread가 읽은 binlog 파일 offset
Relay_Log_File          : mysql-db-replica-relay-bin.000002 -- 사용중인 relaylog
Replica_IO_Running      : Yes -- source에서 binlog 정상 수신 중
Replica_SQL_Running     : Yes -- relay log 정상 적용 중
Exec_Source_Log_Pos     : 805 -- SQL thread가 적용한 binlog 파일 offset
Seconds_Behind_Source   : 0 -- replication lag 없음(완전 동기화)
Source_UUID             : aaa902cd-ecb6-11f0-bda3-000c2921ba33
Retrieved_Gtid_Set      : aaa902cd-ecb6-11f0-bda3-000c2921ba33:65-66 -- source에서 65~66 트랜잭션을 수신
Executed_Gtid_Set       : 82141e86-396c-11f1-8939-000c29ecd7e8:1,
                          aaa902cd-ecb6-11f0-bda3-000c2921ba33:1-66
```
## 4. replica에서 relay log 확인
* 현재 실습에서는 트랜잭션 이벤트가 이미 다 처리된 상태임을 의미
```sql
show relaylog events in 'mysql-db-replica-relay-bin.000002'; -- 현재 사용중인 relay log 확인
```
```sql
+-----------------------------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
| Log_name                          | Pos  | Event_type     | Server_id | End_log_pos | Info                                                               |
+-----------------------------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
| mysql-db-replica-relay-bin.000002 |    4 | Format_desc    |         2 |         126 | Server ver: 8.0.45-0ubuntu0.24.04.1, Binlog ver: 4                 |
| mysql-db-replica-relay-bin.000002 |  126 | Previous_gtids |         2 |         157 |                                                                    |
| mysql-db-replica-relay-bin.000002 |  157 | Rotate         |         1 |           0 | mysql-bin.000016;pos=4                                             |
| mysql-db-replica-relay-bin.000002 |  204 | Format_desc    |         1 |         126 | Server ver: 8.0.45-0ubuntu0.24.04.1, Binlog ver: 4                 |
| mysql-db-replica-relay-bin.000002 |  326 | Rotate         |         0 |           0 | mysql-bin.000016;pos=253                                           |
| mysql-db-replica-relay-bin.000002 |  373 | Gtid           |         1 |         332 | SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:65' |
| mysql-db-replica-relay-bin.000002 |  452 | Query          |         1 |         406 | BEGIN                                                              |
| mysql-db-replica-relay-bin.000002 |  526 | Table_map      |         1 |         454 | table_id: 92 (db1.t1)                                              |
| mysql-db-replica-relay-bin.000002 |  574 | Write_rows     |         1 |         498 | table_id: 92 flags: STMT_END_F                                     |
| mysql-db-replica-relay-bin.000002 |  618 | Xid            |         1 |         529 | COMMIT /* xid=20 */                                                |
| mysql-db-replica-relay-bin.000002 |  649 | Gtid           |         1 |         608 | SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:66' |
| mysql-db-replica-relay-bin.000002 |  728 | Query          |         1 |         682 | BEGIN                                                              |
| mysql-db-replica-relay-bin.000002 |  802 | Table_map      |         1 |         730 | table_id: 92 (db1.t1)                                              |
| mysql-db-replica-relay-bin.000002 |  850 | Write_rows     |         1 |         774 | table_id: 92 flags: STMT_END_F                                     |
| mysql-db-replica-relay-bin.000002 |  894 | Xid            |         1 |         805 | COMMIT /* xid=35 */  
+-----------------------------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
15 rows in set (0.00 sec)  
```
* 메타 이벤트
    * `Format_desc`
        * 해당 relay log 파일의 포맷 정의
        * binlog를 어떻게 해석할지에 대한 헤더
        * 항상 존재하는 파일 시작 메타 정보
    * `Previous_gtids`
        * 해당 relay log가 시작될 때 기준이 되는 GTID 상태
        * 이전까지 어떤 트랜잭션이 있었는지 기록
        * GTID 기반 replication에서는 항상 존재
    * `Rotate`
        * 다음 relay log를 가리킴
* 트랜잭션 흐름 이벤트
    * `GTID` : 트랜잭션 식별자
    * `Query` : 트랜잭션 시작(BEGIN)
    * `Xid` : 커밋, 트랜잭션 종료
    * `Anonymous_GTID` : GTID를 사용하지 않는 환경에서 나타남
* 데이터 변경 이벤트
    * `Table_map` : 어떤 테이블에 대한 작업인지 매핑. 데이터 변경 이벤트 전에 항상 등장
    * `Write_rows` : insert
    * `Update_rows` : update
    * `Delete_rows` : delete 
## 5. lag 실험
### [replica] - SQL thread 중지
```sql
stop replica sql_thread;
```
### [source] - 트랜잭션 수행
```sql
insert into t1 values (6, 600);
insert into t1 values (7, 700);
```
### [replica] - 수신 및 적용 확인
```sql
show replica status\G
```
```sql
Source_Host             : 192.168.111.129
Source_User             : binlog_reader
Source_Log_File         : mysql-bin.000016
Read_Source_Log_Pos     : 1357
Relay_Log_File          : mysql-db-replica-relay-bin.000002
Replica_IO_Running      : Yes -- IO thread는 정상 동작 중. 계속 binlog 수신
Replica_SQL_Running     : No -- SQL thread는 멈춰 있음. binlog 적용하지 않음
Exec_Source_Log_Pos     : 805 -- 수신한 binlog offset과 다름. backlog 존재
Seconds_Behind_Source   : NULL -- SQL thread가 멈춰 있어서 계산 불가
Source_UUID             : aaa902cd-ecb6-11f0-bda3-000c2921ba33
Retrieved_Gtid_Set      : aaa902cd-ecb6-11f0-bda3-000c2921ba33:65-68 -- source에서 트랜잭션 65~68을 수신
Executed_Gtid_Set       : 82141e86-396c-11f1-8939-000c29ecd7e8:1,
                          aaa902cd-ecb6-11f0-bda3-000c2921ba33:1-66 -- replica에서는 트랜잭션 66까지만 적용됨
```
### [replica] - relay log 확인
```sql
show relaylog events in 'mysql-db-replica-relay-bin.000002'; -- 현재 사용중인 relay log 확인
```
```sql
+-----------------------------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
| Log_name                          | Pos  | Event_type     | Server_id | End_log_pos | Info                                                               |
+-----------------------------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
| mysql-db-replica-relay-bin.000002 |    4 | Format_desc    |         2 |         126 | Server ver: 8.0.45-0ubuntu0.24.04.1, Binlog ver: 4                 |
| mysql-db-replica-relay-bin.000002 |  126 | Previous_gtids |         2 |         157 |                                                                    |
| mysql-db-replica-relay-bin.000002 |  157 | Rotate         |         1 |           0 | mysql-bin.000016;pos=4                                             |
| mysql-db-replica-relay-bin.000002 |  204 | Format_desc    |         1 |         126 | Server ver: 8.0.45-0ubuntu0.24.04.1, Binlog ver: 4                 |
| mysql-db-replica-relay-bin.000002 |  326 | Rotate         |         0 |           0 | mysql-bin.000016;pos=253                                           |
| mysql-db-replica-relay-bin.000002 |  373 | Gtid           |         1 |         332 | SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:65' |
| mysql-db-replica-relay-bin.000002 |  452 | Query          |         1 |         406 | BEGIN                                                              |
| mysql-db-replica-relay-bin.000002 |  526 | Table_map      |         1 |         454 | table_id: 92 (db1.t1)                                              |
| mysql-db-replica-relay-bin.000002 |  574 | Write_rows     |         1 |         498 | table_id: 92 flags: STMT_END_F                                     |
| mysql-db-replica-relay-bin.000002 |  618 | Xid            |         1 |         529 | COMMIT /* xid=20 */                                                |
| mysql-db-replica-relay-bin.000002 |  649 | Gtid           |         1 |         608 | SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:66' |
| mysql-db-replica-relay-bin.000002 |  728 | Query          |         1 |         682 | BEGIN                                                              |
| mysql-db-replica-relay-bin.000002 |  802 | Table_map      |         1 |         730 | table_id: 92 (db1.t1)                                              |
| mysql-db-replica-relay-bin.000002 |  850 | Write_rows     |         1 |         774 | table_id: 92 flags: STMT_END_F                                     |
| mysql-db-replica-relay-bin.000002 |  894 | Xid            |         1 |         805 | COMMIT /* xid=35 */                                                |
| mysql-db-replica-relay-bin.000002 |  925 | Gtid           |         1 |         884 | SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:67' |
| mysql-db-replica-relay-bin.000002 | 1004 | Query          |         1 |         958 | BEGIN                                                              |
| mysql-db-replica-relay-bin.000002 | 1078 | Table_map      |         1 |        1006 | table_id: 92 (db1.t1)                                              |
| mysql-db-replica-relay-bin.000002 | 1126 | Write_rows     |         1 |        1050 | table_id: 92 flags: STMT_END_F                                     |
| mysql-db-replica-relay-bin.000002 | 1170 | Xid            |         1 |        1081 | COMMIT /* xid=38 */                                                |
| mysql-db-replica-relay-bin.000002 | 1201 | Gtid           |         1 |        1160 | SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:68' |
| mysql-db-replica-relay-bin.000002 | 1280 | Query          |         1 |        1234 | BEGIN                                                              |
| mysql-db-replica-relay-bin.000002 | 1354 | Table_map      |         1 |        1282 | table_id: 92 (db1.t1)                                              |
| mysql-db-replica-relay-bin.000002 | 1402 | Write_rows     |         1 |        1326 | table_id: 92 flags: STMT_END_F                                     |
| mysql-db-replica-relay-bin.000002 | 1446 | Xid            |         1 |        1357 | COMMIT /* xid=39 */                                                |
+-----------------------------------+------+----------------+-----------+-------------+--------------------------------------------------------------------+
25 rows in set (0.00 sec)
```
* binlog 이벤트는 relay log에 정상적으로 기록됨
* `805` 위치 이후에 해당하는 binlog 이벤트는 아직 적용되지 않음
* RBR(Row-Based Replication)에서는 SQL 문장이 직접 기록되지 않음
* row 이벤트를 SQL 형태로 해석하기 위해서는 추가적인 작업이 필요
### [replica] - 적용되지 않은 트랜잭션의 SQL 확인
```bash
# 805 이후 시작 위치인 925부터 조회
sudo mysqlbinlog -vv --start-position=925 /var/lib/mysql/mysql-db-replica-relay-bin.000002
```
```bash
# 해석에 필요한 부분만 발췌

SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:67'/*!*/;
### INSERT INTO `db1`.`t1`
### SET
###   @1=6 /* INT meta=0 nullable=1 is_null=0 */
###   @2=600 /* INT meta=0 nullable=1 is_null=0 */
COMMIT/*!*/;
# ----------------------------------------------------------------------------
SET @@SESSION.GTID_NEXT= 'aaa902cd-ecb6-11f0-bda3-000c2921ba33:68'/*!*/;
### INSERT INTO `db1`.`t1`
### SET
###   @1=7 /* INT meta=0 nullable=1 is_null=0 */
###   @2=700 /* INT meta=0 nullable=1 is_null=0 */
COMMIT/*!*/;
```
### [replica] - SQL thread 재시작
```sql
start replica sql_thread;
```
### [replica] - backlog가 해소되는지 확인
```sql
show replica status\G
```
```sql
Source_Host             : 192.168.111.129
Source_User             : binlog_reader
Source_Log_File         : mysql-bin.000016
Read_Source_Log_Pos     : 1357
Relay_Log_File          : mysql-db-replica-relay-bin.000002
Replica_IO_Running      : Yes 
Replica_SQL_Running     : Yes -- SQL thread는 정상 동작 중
Exec_Source_Log_Pos     : 1357 -- 수신한 binlog offset과 같아짐
Seconds_Behind_Source   : 0 -- replication lag 없음(완전 동기화)
Source_UUID             : aaa902cd-ecb6-11f0-bda3-000c2921ba33
Retrieved_Gtid_Set      : aaa902cd-ecb6-11f0-bda3-000c2921ba33:65-68
Executed_Gtid_Set       : 82141e86-396c-11f1-8939-000c29ecd7e8:1,
                          aaa902cd-ecb6-11f0-bda3-000c2921ba33:1-68 -- source에서 읽어온 트랜잭션과 적용한 트랜잭션이 68로 같아짐
```