# ★Recovery 공부 시 다시 확인하고 이어서 정리하기★

# 트랜잭션(Transaction)이란?
# Undo Log
* 트랜잭션의 `원자성(Atomicity)`를 보장하기 위한 핵심 메커니즘
* 트랜잭션 수행 중 문제가 발생하거나 롤백 요청 시, 이전 상태로 복원하기 위해 사용
* 데이터 변경을 취소하고 원래 상태로 되돌림
* MVCC의 Read Consistency(읽기 일관성)을 구현하는 핵심 요소
* 구조
    * 하나의 트랜잭션에서 발생한 여러 undo log는 `체인`(연결 리스트) 형태로 관리됨  
        * 각 트랜잭션이 변경한 데이터에 대한 undo log가 순서대로 연결되어 있음
        * 롤백 시 마지막 변경부터 거꾸로 순서대로 이전 상태로 복구
    * 연결된 여러개의 undo log 들은 `undo log segment` 단위로 관리됨
    * undo log segment는 `undo tablespace` 내에 저장됨
        * undo tablespace는 일반 데이터 파일과 분리되어 관리됨
* 저장 되는 내용 (Undo Log Entry)
    * 트랜잭션 ID
    * undo log 타입 : `INSERT Undo`, `DELETE Undo`, `UPDATE Undo`, `DDL Undo`, `TRUNCATE Undo`
    * 변경되기 전의 데이터 값
    * undo log의 순서 및 이전 undo log 포인터
    * 원본 데이터 위치 (rowid, 페이지 번호 등)
<br>

# Redo Log
* 트랜잭션의 `영속성(Durability)`을 보장하기 위한 핵심 메커니즘
* DB 서버가 비정상 종료되었을 경우, 디스크에 반영되지 못한 변경 사항을 다시 재생(redo) 해서 복구
* 구조
    * 하나의 페이지 내 변경마다 하나의 `redo log record`를 생성
        * 트랜잭션 하나 당 redo log record 하나 아님
        * SQL문 하나 당 redo log record 하나 아님
        * **SQL문이 건드리는 페이지 하나 당 redo log record 하나**
            * 단일 SQL문에서 하나 또는 여러 페이지를 변경할 수 있음
            * 즉, 단일 SQL문에서 하나 또는 여러 개의 redo log record가 생성될 수 있음
    * redo log record는 순환 구조로 저장됨
* `WAL(Write-Ahead Logging)` 원칙을 따름
    * 데이터 페이지를 디스크에 쓰기 전에 반드시 redo log를 먼저 디스크에 기록
        1. 변경 내용(redo log)을 메모리(`redo log buffer`)에 기록
        2. 커밋 시, redo log buffer의 내용을 디스크(`redo log file`)로 flush
        3. 나중에 메모리(`buffer pool`)의 dirty page를 디스크(`data file`)에 반영
            * dirty page : 메모리에만 있고 디스크에는 아직 반영되지 않은 실제 데이터 페이지
            * redo log를 참조하지 않고 메모리의 데이터 페이지를 디스크에 그대로 기록
    * 정상 동작 시, redo log는 기록되기만 할 뿐 사용되지는 않음
    * 장애 복구 시, 디스크에 저장된 redo log를 재생하여 data file을 복구
* 저장 되는 내용 (Redo Log Record)
    * 위치 정보
        * tablespace id
        * page id
    * 페이지 내 변경 위치
        * offset
        * 길이
    * 변경 내용
        * 변경된 값
        * 변경 방법
    * 관리 정보
        * redo log 번호(LSN)
        * 트랜잭션 ID
        * checksum 등
# MVCC (Multi-Version Concurrency Control)
# 동시성 제어
# 트랜잭션 복구 메커니즘