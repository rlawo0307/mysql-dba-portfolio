# 목차
* 참조 무결성 검증
* restrict/cascade 옵션 동작 확인
* 인덱스 동작
    * FK 생성 시 인덱스 자동 생성 확인
    * FK 삭제 후 인덱스 유지 확인
* 참고 자료
    * [→ fk에 대한 개념 자세히 보기](../constraint-theory.md)
<br><br>

# 참조 무결성 검증
### table 생성
```sql
create table parent (pk int primary key);
create table child (c1 int, fk int, foreign key(fk) references parent(pk)); -- default : restrict

insert into parent values (1);
```
### child insert
* child table의 fk 값에 대해 parent table에 해당 row가 존재하는지 확인
```sql
insert into child values (100, 1); -- success
insert into child values (100, 2); -- error 1452
```
```sql
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`db1`.`child`, CONSTRAINT `child_ibfk_1` FOREIGN KEY (`fk`) REFERENCES `parent` (`pk`))
```
### parent update/delete
* 해당 parent row를 child table이 참조하고 있는지 확인
```sql
update parent set pk = 2 where pk = 1; -- error 1451
delete from parent where pk = 1; -- error 1451
```
```sql
ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`db1`.`child`, CONSTRAINT `child_ibfk_1` FOREIGN KEY (`fk`) REFERENCES `parent` (`pk`))
```
<br>

# restrict/cascade 옵션 동작 확인
### restrict 옵션
```sql
create table parent (pk int primary key);
create table child (c1 int, fk int, foreign key(fk) references parent(pk) on delete restrict on update restrict);

insert into parent values (1);
insert into child values (100, 1);

delete from parent where pk = 1; -- ERROR 1451
update parent set pk = 2 where pk = 1; -- ERROR 1451
-- child table에서 parent row를 참조하고 있으므로 parent row의 update 및 delete가 모두 거부됨
```
```sql
ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`db1`.`child`, CONSTRAINT `child_ibfk_1` FOREIGN KEY (`fk`) REFERENCES `parent` (`pk`) ON DELETE RESTRICT ON UPDATE RESTRICT)
```
### cascade 옵션
```sql
create table parent (pk int primary key);
create table child (c1 int, fk int, foreign key(fk) references parent(pk) on delete cascade on update cascade);

insert into parent values (1);
insert into child values (100, 1);
```
* update 수행
```sql
select parent.pk, child.fk from parent join child on parent.pk = child.fk; -- before
update parent set pk = 3 where pk = 1; -- parent 행을 변경
select parent.pk, child.fk from parent join child on parent.pk = child.fk; -- after
```
```sql
[ BEFORE ]              [ AFTER ]
+----+------+           +----+------+
| pk | fk   |           | pk | fk   |
+----+------+     →     +----+------+
|  1 |    1 |           |  3 |    3 |
+----+------+           +----+------+
```
* delete 수행
```sql
delete from parent where pk = 3;
select parent.pk, child.fk from parent join child on parent.pk = child.fk;
```
```sql
Empty set (0.00 sec)
-- parent 삭제 시 해당 row를 참조하는 child row도 cascade에 의해 자동으로 삭제됨
```
<br>

# FK 생성 시 인덱스 자동 생성 확인
### parent table 생성
```sql
create table parent (pk int primary key);
```
### child table에 인덱스 없음 
```sql
create table child (c1 int, fk int, foreign key(fk) references parent(pk));
show index from child;
```
```sql
Table           : child
Key_name        : fk -- 컬럼명과 동일한 인덱스를 자동으로 생성
Seq_in_index    : 1
Column_name     : fk
```
### child table에 단일 인덱스 존재
```sql
create table child1 (c1 int, fk int,
                    index idx(fk), 
                    foreign key(fk) references parent(pk));
show index from child1;
```
```sql
Table           : child
Key_name        : idx -- 기존에 생성한 인덱스
Seq_in_index    : 1
Column_name     : fk
```
### child table에 복합 인덱스 존재 + left most 컬럼이 fk
```sql
create table child2 (c1 int, fk int,
                    index idx(fk, c1),
                    foreign key(fk) references parent(pk));
show index from child2;
```
```sql
Table           : child2
Key_name        : idx -- 기존에 생성한 인덱스
Seq_in_index    : 1
Column_name     : fk

Table           : child2
Key_name        : idx -- 기존에 생성한 인덱스
Seq_in_index    : 2
Column_name     : c1
```
### child table에 복합 인덱스 존재 + left most 컬럼이 fk 아님
```sql
create table child3 (c1 int, fk int,
                    index idx(c1, fk),
                    foreign key(fk) references parent(pk));
show index from child3;
```
```sql
Table           : child3
Key_name        : idx -- 기존에 생성한 인덱스
Seq_in_index    : 1
Column_name     : c1

Table           : child3
Key_name        : idx -- 기존에 생성한 인덱스
Seq_in_index    : 2
Column_name     : fk

Table           : child3
Key_name        : fk -- 컬럼명과 동일한 인덱스를 자동으로 생성
Seq_in_index    : 1
Column_name     : fk
```
## 결과
* 인덱스가 없는 경우 → 컬럼명과 동일한 인덱스를 자동으로 생성
* 단일 인덱스 존재 → 기존 인덱스 재사용
* 복합 인덱스 존재
    * left most가 fk인 경우 → 기존 인덱스 재사용
    * left most가 fk가 아닌 경우 → 컬럼명과 동일한 인덱스를 자동으로 생성
## 결론
child table의 fk 컬럼에 인덱스가 없거나, 인덱스가 존재하지만 fk 컬럼이 left-most 위치에 있지 않아 사용할 수 없는 경우, MySQL이 자동으로 인덱스를 생성하는 것을 확인할 수 있다.

InnoDB에서 fk를 생성할 때, child table의 fk 컬럼에 대해 참조 무결성 검증을 위해 인덱스가 반드시 필요하다. fk 검증 과정에서 child → parent 조회와 parent → child 참조 여부 확인이 index lookup 기반으로 수행되기 때문이다. (** 단, fk 검증 과정은 InnoDB 엔진 내부에서 수행되므로 explain 또는 explain analyze에서 직접 확인할 수 없다.**)

따라서 fk 컬럼은 인덱스의 left-most 컬럼이어야 하며, 이를 만족하는 인덱스가 존재하지 않을 경우 InnoDB는 자동으로 인덱스를 생성한다.
<br>

# fk 삭제 시 인덱스 자동 삭제 확인
```sql
create table parent (pk int primary key);
create table child (c1 int, fk int, constraint fk_child_parent foreign key(fk) references parent(pk));

show index from child; -- before
alter table child drop foreign key fk_child_parent;
show index from child; -- after
```
```sql
-- before
Table           : child
Key_name        : fk_child_parent
Seq_in_index    : 1
Column_name     : fk

-- after
Table           : child
Key_name        : fk_child_parent
Seq_in_index    : 1
Column_name     : fk
```
## 결과 및 결론
fk 제약을 삭제한 이후에도 child table의 fk 컬럼에 생성된 인덱스는 삭제되지 않고 유지된다. 따라서 fk 제약 제거 이후 불필요한 인덱스가 남을 수 있으며, 필요에 따라 별도로 정리해야 한다.
<br>