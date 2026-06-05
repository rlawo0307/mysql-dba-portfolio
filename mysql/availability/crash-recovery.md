# Crash Recovery란?
* 비정상 종료(crash) 이후 데이터의 일관성을 복구하는 과정
* MySQL InnoDB는 crash 발생 시 자동으로 recovery 수행
    * DBA가 직접 restore를 수행하지 않음
    * 서버 시작 과정에서 InnoDB가 수행
* crash 발생 과정
    * InnoDB는 성능을 위해 모든 변경 내용을 즉시 data file에 기록하지 않음
        * `buffer pool 수정 → redo log 기록 → commit → 나중에 data file에 기록`
    * 커밋 후 data file 반영 전 crash 발생 가능
        * buffer pool은 휘발됨
        * data file에는 커밋 전 데이터만 남아있음
## Crash Recovery 메커니즘
* 커밋된 변경 내용 복구
    * `redo log`
        * crash recovery를 위해 데이터 변경 내역을 기록하는 로그
        * 커밋된 변경 내용을 복구하기 위해 사용
    * `WAL`
        * 커밋 전, data file보다 redo log를 먼저 기록하는 저장 메커니즘
    * `checkpoint`
        * redo log 내용이 data file에 반영된 시점
        * crash recovery 시작 위치를 결정
        * ex) redo log : 100 ~ 104, checkpoint : 102
            * 100 ~ 102 는 이미 data file에 반영 완료
            * recovery 시 103 이후만 재적용
* 커밋되지 않은 변경 내용 처리
    * `undo log`
        * rollback과 MVCC를 지원하기 위해 변경 이전 데이터를 저장하는 로그
        * 미완료 트랜잭션을 rollback하기 위해 사용
### crash recovery 과정
```text
0. crash 발생
1. MySQL 재시작
2. checkpoint 확인 → 어디까지 data file에 반영되었는지 확인
3. redo log 스캔   → checkpoint 이후의 변경 내용 확인
4. redo log 적용   → 커밋된 변경 내용 복구
5. undo log 적용   → 미완료 트랜잭션 rollback
6. recovery 완료
```
## Crash Recovery 시간이 길어지는 경우
* redo log가 너무 많은 경우
* dirty page가 과도한 경우
* 대용량 update 수행 중 장애 발생
* 장시간 checkpoint가 진행되지 않을 경우