```text
[참고]

Replication : 한 서버의 변경 사항을 다른 서버들에 복제
Clustering : 복제 + 고가용성 운영 구조(서비스 지속성, 노드 집합 관리)
```
<br>

# Replication이란?
* source 서버의 데이터를 하나 이상의 replica 서버로 복사하는 기술
    * 기존에는 master, slave 용어 사용
    * 현재 공식 권장 용어는 source, replica
* 기본 동작은 비동기
* replica는 source와 항상 계속 연결되어 있지 않아도 됨
* 범위
    * 전체 DB
    * 특정 DB
    * 특정 테이블
* 사용 목적
    * 읽기 트래픽 분산
    * 백업 서버 운영
    * 장애 발생 시 복구용 replica 확보
    * 분석/리포트용 서버 분리
    * 지역 간 데이터 복제
### 동작 원리
```text
Client
  │
  ▼
Source
  1. 트랜잭션 실행
  2. Binary Log(binlog)에 변경 사항 기록
  3. Binlog Dump Thread가 Replica로 binlog 전송
         │
         ▼
Replica
  1. Receiver(I/O) Thread가 수신
  2. Relay Log에 저장
  3. SQL Thread가 실행하여 반영
```
* `Binary Log`
    * source에서 발생한 변경 이벤트 기록
    * 단순히 data file을 통째로 실시간 복사하는 것이 아님
    * 변경 이력(이벤트/로그) 기반으로 동작
* `Relay Log`
    * replica가 source로부터 받은 이벤트를 임시 저장하는 로그
### Replication 종류
* Synchronous Replication
    * source는 트랜잭션 커밋 전에 모든 replica 변경사항이 적용(커밋)될 때가지 기다림
    * **MySQL은 완전한 synchronous replication을 지원하지 않음**
        * 대신 semi-sync, group replication를 통해 유사한 동작 제공
* `Asynchronous Replication`
    * 특징
        * source는 변경사항이 replica에 반영될 때까지 기다리지 않음
        * source는 커밋 후 즉시 응답
        * 이후 replica로 전파
        * MySQL replication 기본 동작
    * 장점
        * 성능 상 유리
        * 구조가 단순함
    * 단점
        * replication lag가 발생할 수 있음
        * replica에서 read-after-write 일관성이 보장되지 않음
            * 내가 방금 쓴 데이터를 바로 읽는 작업
        * source 장애 직전에 아직 replica에 전달되지 않은 트랜잭션은 유실될 수 있음
        * 자동 [failover](failover.md) 없음
    * 주로 사용되는 상황
        * 읽기 트래픽이 많을 경우
        * 쓰기는 한 곳에서만 처리할 경우
        * 약간의 replication lag는 괜찮은 경우
        * 장애 시 데이터 유실이 치명적이지 않은 경우
        * 애플리케이션 구조를 단순하게 유지하고 싶을 경우
* `Semi-synchronous Replication`
    * 특징
        * source는 최소 하나의 replica가 이벤트를 수신했다는 ACK를 기다림
        * replica의 실제 데이터 반영 완료 여부는 보장하지 않음
    * 장점
        * async보다 내구성 향상
    * 단점
        * 모든 노드가 완전히 같은 상태를 즉시 보장하지 않음
    * 주로 사용되는 상황
        * 데이터 유실을 최소화하고 성능이 중요한 경우
        * async는 불안하고 완전 동기식은 부담될 경우
* `Group Replication`
    * 특징
        * 고가용성 그룹을 구성하기 위한 복제 방법
        * 여러 서버가 하나의 그룹을 형성
        * 그룹 멤버십을 기반으로 노드 상태 관리
        * 트랜잭션 커밋 전에 그룹 내에서 인증 과정 수행
        * 다른 노드와 충돌 여부 확인 후 커밋
        * virtually synchronous 방식
    * 장점
        * single/multi primary 모드 지원
            * write node가 하나 → single primary mode
            * write node가 여러개 → multi primary mode
        * 장애 감지에 탁월함
        * 장애 시 자동 [failover](failover.md) 가능
        * async보다 더 강한 데이터 일관성 보장
    * 단점
        * 구조가 복잡함
        * 성능 오버헤드가 있음
        * 운영 난이도 높음
        * multi primary 모드의 경우, 설계 난이도가 매우 높고 write 충돌 관리가 어려움
### Replication Lag
* source의 커밋 시점과 replica의 반영 시점 사이에서 발생하는 시간 차이
* replica가 source를 늦게 따라가는 상태
* async replication에서 발생
    * source는 replica의 반영을 기다리지 않음
    * source는 트랜잭션이 커밋되었지만
    * replica에 변경사항이 반영되었는지 보장할 수 없음
* 발생 원인
    * 네트워크 지연
    * replica의 apply 속도 부족
    * 단일 apply 쓰레드 병목
    * 무거운 DDL/DML
    * 대량 쓰기 작업
    * replica의 CPU/IO 부족
    * 긴 lock 대기
* 영향
    * source와 replica 간 데이터 불일치가 발생할 수 있음
    * read-after-write 일관성 깨질 수 있음
    * [failover](failover.md) 시, lag 큰 replica가 source로 승격되면 아직 반영되지 않은 데이터가 유실될 수 있음
* 대응 방법
    * write 직후 read 요청을 source로 라우팅
    * write한 사용자로부터 발생한 read 요청은 일정 시간동안 source로 라우팅 → `stickiness 적용`
    * replica 성능 개선
    * 무거운 작업 운영 트래픽과 분리해서 수행
<br><br>

# Clustering이란?
* 여러 노드가 하나의 고가용성 시스템처럼 동작하도록 구성
* 단순 복제뿐만 아니라 서비스 지속성과 노드 집합 관리가 더해진 형태
* replication이 "데이터 복제"라면, clustering은 "서비스 전체를 유지하는 구조"
* clustering은 고가용성(HA)를 위한 핵심 구조
### Clustering 종류
* group replication 기반 cluster
    * replication을 이용한 고가용성 구조
    * 일반 MySQL 서버를 기반으로 구성
    * 여러 MySQL 서버가 하나의 그룹을 형성
        * `single-primary mode`
        * `multi-primary mode`
    * 그룹 멤버십을 기반으로 노드 상태 관리
        * 트랜잭션 커밋 전에 그룹 내에서 인증 과정 수행
        * 다른 노드와 충돌 여부 확인 후 커밋
    * 자동 [failover](failover.md) 가능
* `NDB cluster`
    * 특징
        * 분산 데이터베이스 구조
        * shared-nothing 아키텍처
            * 각 노드가 서로 아무것도 공유하지 않음
            * 각 서버가 독립적인 CPU, 메모리, 디스크를 가지고 있음
        * 노드 간 데이터 분산
        * 메모리 중심 구조
    * 구성
        * data node : 실제 NDB 데이터 저장
        * management node : 클러스터 설정/관리
        * sql node : 클라이언트가 접속하는 MySQL 서버 계층
    * 장점
        * 높은 가용성
        * 수평 확장(scale-out) 가능
        * 노드 장애 시에도 일부 서비스 유지 가능
    * 단점
        * 일반 InnoDB 기반 MySQL 구조와 다름
        * 설계 및 운영 난이도 매우 높음
        * 메모리 중심 구조로 비용 증가
        * 일부 기능 제한 존재