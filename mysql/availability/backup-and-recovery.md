```text
[참고]

backup(백업)      : 복구를 위해 데이터를 보존하는 작업
restore(복원)     : 백업 데이터를 DB에 다시 적용하는 작업 
recovery(복구)    : restore를 포함하여 DB를 정상 상태로 되돌리는 전체 과정

※ restore ⊂ recovery
```
# Backup이란?
* 장애, 데이터 손상, 사용자 실수 등에 대비하여 데이터를 복구할 수 있도록 데이터를 보관하는 작업
* DB 운영에서 장애 복구를 위한 핵심 수단
* 특정 시점의 데이터를 보존하여 필요 시 원하는 복구 지점까지 복구 가능
## Backup 분류
### 저장 방식 기준
* `logical backup`
    * 데이터를 sql 또는 논리적 형태로 저장하는 방식
    * 대표 도구 : `mysqldump`
    * 백업 대상
        * schema : database, table, view
        * data : row data
        * object : trigger, routine, event (옵션에 따라 포함 여부가 달라질 수 있음)
    * 특징
        * sql 재실행 방식으로 복구
        * 사람이 읽을 수 있음
    * 장점
        * 사람이 직접 내용 확인 가능
        * 특정 database / table / object 단위 선택 백업 가능
        * physical backup보다 부분 복구가 쉬움
        * 물리 파일 구조 의존성이 낮음
        * OS / storage 환경 차이에 덜 민감함
        * 다른 서버나 환경으로 migration에 유리
    * 단점
        * 대용량 DB에서는 백업 속도가 느릴 수 있음
        * 복구 시 sql 재실행이 필요하므로 시간이 오래 걸림
        * 백업 중 일관성 확보를 위한 옵션 선택이 중요함
* `physical backup`
    * DB가 사용하는 물리 파일 자체를 백업하는 방식
    * 대표 도구 : `file copy`, `Percona XtraBackup`, `MySQL Enterprise Backup`
    * 백업 대상
        * data file
        * redo log
        * undo file
        * binary log
        * relay log (환경에 따라)
        * configuration file (운영 정책에 따라)
    * 특징
        * sql 재실행 없이 파일 자체를 복구
    * 장점
        * 백업 및 복구 속도가 빠름
        * 대용량 환경에 적합
        * RTO가 짧은 환경에 유리
    * 단점
        * 사람이 읽을 수 없음
        * 부분 복구가 어려움
        * storage 구조 의존성이 높음
        * 일관성 확보 절차가 중요함
### 범위 기준
* `full backup`
    * 전체 데이터를 모두 백업
    * 복구가 단순함
    * 백업 시간과 저장 공간이 많이 필요
* `incremental backup`
    * 마지막 백업 이후 변경된 데이터만 백업
    * 백업 속도와 저장 공간 효율이 좋음
    * 복구 과정이 복잡함
    * tool에 따라 지원 여부가 다름
* `differential backup`
    * 마지막 full backup 이후 변경된 데이터만 누적 백업
    * incremental보다 복구가 단순함
    * 시간이 지날수록 백업 크기 증가
### 서비스 중단 여부 기준
* `hot backup`
    * 서비스 중단 없이 수행 가능한 백업
    * read / write workload와 동시에 진행 가능
* `warm backup`
    * 일부 기능을 제한한 상태에서 백업
    * 일반적으로 write 제한 또는 lock 필요
* `cold backup`
    * DB 중지 후 백업
    * 가장 단순하고 일관성 확보가 쉬움
    * 서비스 중단 필요
## Backup 전략 설계 요소
* `RPO`(Recovery Point Objective)
    * 허용 가능한 데이터 손실 범위
    * 장애 발생 시 얼마나 최근 시점까지 복구해야 하는지를 의미
    * RPO가 작을수록 더 자주 복구 지점을 확보할 수 있는 백업 전략 필요
    * ex)RPO = 24시간 → 최악의 경우 최대 24시간 데이터 손실 허용
* `RTO`(Recovery Time Objective)
    * 허용 가능한 서비스 중단 시간
    * 장애 발생 시 얼마나 빨리 복구해야 하는지를 의미
    * RTO가 짧을수록 빠른 복구가 필요
    * ex) RTO = 10분 → 10분 안에 복구해야 함
## 기타 고려 사항
* 데이터 크기
* 서비스 downtime 허용 여부
* 백업 저장 공간
* 백업 실행 시간
* 복구 검증 주기 등
<br><br>

# Recovery란?
* 장애 발생 후 DB를 정상 상태로 되돌리는 복구 과정
* 백업 데이터를 이용한 restore 또는 변경 이력 재적용 등을 포함할 수 있음
## 종류
* `full restore`
    * 전체 DB를 복원
* `partial restore`
    * 특정 database / table / object만 복원
* `PITR`(Point-in-Time Recovery)
    * full backup 이후 변경 내용을 재적용하여 원하는 특정 시점까지 복구
    * MySQL에서는 full backup + binary log를 사용
## 복구 절차
```text
1. 장애 원인 파악
2. 복구 목표 시점 결정 (RPO 기준 반영)
3. 사용할 백업 확인
4. restore 수행
5. PITR의 경우, binary log 재적용 필요
6. 데이터 검증
    * backup 성공 ≠ restore 성공 ≠ recovery 성공
    * 백업이 정상 생성되었더라도 실제 restore가 실패할 수 있음
    * restore가 성공했더라도 데이터 정합성 검증이나 서비스 정상화에 실패하면 전체 recovery는 실패할 수 있음
7. 서비스 재개
```
## 고려 사항
* RTO 충족 가능 여부
* 백업 무결성
* restore 시간
* 데이터 정합성
* partial restore 가능 여부
* 운영 영향도