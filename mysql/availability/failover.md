# Failover란?
* 장애가 발생한 source를 대신해 다른 노드가 그 역할을 이어받는 과정
* 서비스 중단 시간을 줄이고 쓰기 가능 상태를 복구하는 작업
## 동작 원리
```text
1. 기존 source에 장애 발생
2. 장애 감지 (정말 장애가 발생했는지 판단)
3. 승격할 replica 선정
    - 기존 source를 가장 많이 따라온 replica
    - replication lag가 적은가
    - 복제 thread가 정상인가
    - 승격 후 write 처리가 가능한 상태인가
4. 선택한 replica를 source로 승격
    - read only 해제
    - write 가능한 상태로 전환
5. 애플리케이션 write 트래픽을 새 source로 전환
6. 나머지 replica도 새 source를 따라가도록 구성
7. 기존 source 처리
    - 즉시 다시 서비스에 넣지 않음
    - 필요하면 재동기화
    - 재빌드 후 replica로 편입
```
## 주의할 점
* async replication에서는 replica에 아직 반영되지 않은 데이터가 유실될 수 있음
* 잘못된 장애 판단은 split-brain을 유발할 수 있음
* 기존 source가 복구되더라도 바로 다시 붙이면 안됨
    * 새 source와 데이터가 갈라졌을 수 있음
## Failover 종류
* 수동 failover
    * 운영자가 직접 판단해서 failover 수행
    * 휴리스틱하게 확인하여 오판 가능성을 줄일 수 있음
    * 야간 장애 및 즉각적인 대응이 어려움
    * 절차 숙련도 필요
* 자동 failover
    * 시스템이 장애를 감지하고 자동으로 전환
#### replication 방식 별 failover 차이
|                     |             |
|---------------------|:-----------:|
|**async replication**|수동 failover<br>(툴 사용 시 자동화 가능)|
|**semi-replication**|수동 failover|
|**group replication**|자동 failover 지원|
|**NDB cluster**|자동 failover 지원|
## Connection Failover 와 Server Failover
* connection failover
    * replica의 복제 연결 대상을 새 source로 바꾸는 것
* server failover
    * 서비스의 write 역할 자체가 다른 노드로 넘어가는 것
## Failover vs Switchover
* failover
    * 비정상 상황의 전환
    * 장애 때문에 어쩔 수 없이 전환
* switchover
    * 정상 상황의 계획된 전환
    * 점검/업그레이드 등을 위해 계획적으로 전환