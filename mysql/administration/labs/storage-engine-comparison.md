# 목차
* Storage Engine 확인
* Engine 별 transaction 지원 여부 확인
* 참고자료
    * [테이블 생성 및 변경하는 방법(DDL)](../../sql-commands/sql-ddl.md)
<br><br>

# Storage Engine 확인
* 기본 storage engine이 무엇인지 확인해보자
* `InnoDB`, `MyISAM` 등이 사용가능 한지 확인해보자
* transaction 지원 여부가 어떻게 표시되는지 확인해보자
```sql
show engines;
``` 
```sql
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.00 sec)
```
* `Enigne`
    * 스토리지 엔진 이름
    * 테이블이 실제로 데이터를 저장하는 방식
        * InnoDB : 현재 MySQL 기본 엔진
        * MyISAM : 과거 기본 엔진(지금은 거의 안씀)
* `Support`
    * 엔진이 현재 서버에서 사용 가능한 상태인지를 나타냄
        * YES : 사용 가능
        * NO : 지원 안함(비활성 또는 미설치)
        * DEFAULT : 기본 엔진
        * DISABLED : 설정으로 비활성화됨
* `Transaction`
    * 트랜잭션 지원 여부
* `XA`
    * 분산 트랜잭션(2-phase commit) 지원 여부
* `Savepoints`
    * savepoint 지원 여부
    * 트랜잭션 내부에서 부분 rollback 가능 여부
## 결론
* InnoDB만 트랜잭션, XA, savepoints를 모두 지원
* MyISAM은 트랜잭션을 지원하지 않음
* 대부분의 엔진은 특수 목적용이며 일반적인 OLTP 환경에서는 InnoDB 사용이 기본
<br><br>

# Engine 별 transaction 지원 여부 확인
* rollback 이후에도 값이 유지되는지 확인해보자
* MyISAM은 트랜잭션을 지원하지 않으므로 rollback 되지 않음
## InnoDB
```sql
create table t1(c1 int) engine=InnoDB;
insert into t1 values(1);

start transaction;
update t1 set c1 = 999 where c1 = 1;
rollback;

select * from t1;
```
```sql
+------+
| c1   |
+------+
|    1 |
+------+
1 row in set (0.00 sec)
```
## MyISAM
```sql
create table t2(c1 int) engine=MyISAM;
insert into t2 values(1);

start transaction;
update t2 set c1 = 999 where c1 = 1;
rollback;

select * from t2;
```
```sql
+------+
| c1   |
+------+
|  999 |
+------+
1 row in set (0.00 sec)
```
## 결과
* InnoDB : rollback 후 c1이 초기 값으로 복구됨
* MyISAM : rollback 해도 c1의 값이 변하지 않음
## 결론
InnoDB는 트랜잭션을 지원하며 undo log를 통해 변경 이전 데이터를 보관하기 때문에 rollback이 가능하다. 따라서 rollback 수행 시 데이터는 트랜잭션 시작 이전 상태로 복구된다.

반면, MyISAM은 트랜잭션을 지원하지 않기 때문에 `start transaction`은 의미가 없고 모든 변경은 즉시 반영된다(autocommit처럼 동작). 따라서 rollback을 수행해도 이미 반영된 데이터는 되돌릴 수 없다.

결론적으로, storage engine에 따라 동일한 SQL이라도 동작이 완전히 달라질 수 있으므로 적절한 엔진을 선택하여 사용하는 것이 중요하다. 특히, OLTP 환경에서는 트랜잭션, 동시성 제어, crash recovery가 중요하므로 InnoDB를 사용하는 것이 일반적이다.