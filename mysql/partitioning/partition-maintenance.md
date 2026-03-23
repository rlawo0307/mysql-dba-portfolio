# Partition 관리
* partition은 DDL을 통해 동적으로 관리할 수 있음
* 주로 데이터 증가 및 삭제에 맞춰 partition을 추가하거나 제거하는 용도로 사용
## Add Partition
* 동적으로 새로운 partition 추가하는 작업
* 기존 partition의 데이터에는 영향 없음
* range partition에서 범위를 뒤쪽으로 확장할 때 주로 사용
    * 이 경우, partition 범위는 반드시 기존 범위보다 커야 함
* 기존에 maxvalue partition이 있으면 partition 추가 불가능
    * reorganize로 partition을 분할해야 함
```sql
-- 기존 파티션
create table t1(c1 int, c2 int) partition by range(c1)(
    partition p1 values less than(10),
    partition p2 values less than(20)
);

-- 새로운 파티션 추가
alter table t1 add partition (
    partition p3 values less than(30),
    partition p4 values less than(40),
    partition pmax values less than(maxvalue)
);
```
## Drop Partition
* 특정 partition을 삭제하는 작업
* 해당 partition의 데이터도 함께 삭제됨
* 대용량 데이터 삭제에 효과적
    * 대량 delete보다 더 빠르게 데이터 삭제 가능
* 삭제된 데이터는 복구 불가하므로 주의 필요
```sql
alter table t1 drop partition p1;
```
## Reorganize Partition
* 기존 partition을 재구성하는 작업
* partition을 나누거나 합칠 때 사용
* 특정 범위를 세분화하거나 maxvalue partition을 분할할 때 사용
* 기존 데이터가 새로운 partition으로 재배치됨
```sql
-- partition 나누기
alter table t1 reorganize partition pmax into(
    partition p5 values less than(50),
    partition p6 values less than(60),
    partition pmax values less than(maxvalue)
);

-- partition 합치기
alter table t1 reorganize partition p1, p2 into(
    partition p12 values less than(20)
);
```