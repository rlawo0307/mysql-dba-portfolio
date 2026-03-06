????? 다시 실험 필요
# 목차
* FK 생성 시 인덱스 자동 생성 확인
* ON DELETE 옵션 실험 (CASCADE vs RESTRICT)
* FK 검증 과정 확인
* FK 검증 비용 실험 (대량 child 데이터)
* FK + CASCADE 대량 삭제 성능 실험
* FK Locking 실험
* 참고 자료
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
### child table에 복합 인덱스 존재 + left most 컬럼이 pk
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
### child table에 복합 인덱스 존재 + left most 컬럼이 pk 아님
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
## 결론
# ON DELETE 옵션 실험 (CASCADE vs RESTRICT)
# FK 검증 과정 확인
# FK 검증 비용 실험 (대량 child 데이터)
# FK + CASCADE 대량 삭제 성능 실험
# FK Locking 실험