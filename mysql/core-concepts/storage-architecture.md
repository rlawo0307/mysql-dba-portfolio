# InnoDB 물리적 저장 구조
* InnoDB는 계층적 물리 구조를 가짐
```text
Tablespace
  └─ Segment
       └─ Extent
            └─ Page
                 └─ Record(Row)
```
## Record
* 실제 데이터 한 행(row)
## Page
* InnoDB가 디스크 I/O를 수행하는 최소 단위
* 크기 : `16KB` (변경 가능)
* 버퍼 풀에도 페이지 단위로 적재
* 종류
    * Data Page
        * 실제 row 데이터가 저장되는 페이지
        * data page = `clustered index의 leaf page`
            * 테이블 전용 데이터 페이지가 따로 존재하지 않음
        * pk 기준으로 정렬되어 저장됨
        * insert 시 page split이 발생할 수 있음
        * update 시 row size가 증가하면 page overflow가 발생할 수 있음
        * delete 시 바로 삭제되는 것이 아니라 delete mark를 남김
        * 하나의 sql이 여러 page를 건드릴 수 있음
        * redo log는 변경된 페이지마다 redo record를 생성
    * Index Page
        * B+Tree의 노드를 구성하는 페이지
        * 종류
        * leaf page
            * clustered index leaf page (= data page)
                * pk + 실제 데이터
            * secondary index leaf page
                * secondary key + pk
        * non-leaf page
            * key + child page pointer
    * Undo Page
        * undo log를 저장하는 page [→ undo log에 대해 자세히 보기](../transaction-management.md)
        * 트랜잭션 이전 값을 저장
        * 저장 위치
            * 별도의 undo tablespace
            * rollback segment 내부
    * System Page
        * InnoDB 내부 메타데이터를 저장하는 페이지
        * 공간 할당 및 세그먼트 관리, 트랜잭션 상태 추적에 사용
        * 종류
            * FSP Header Page
                * tablespace 메타 정보 저장
                * extent 상태 관리
            * INODE Page    
                * segment 관리 정보
            * TRX System Page
                * 트랜잭션 시스템 정보
            * Data Dictionary Page
                * 테이블/인덱스 정의 정보
    * IBUF Bitmap Page, BLOB page, Compressed Page 등..
## Extent
* 연속된 64개의 page의 묶음
* 크기 : 16KB * 64 = `1MB`
* segment가 공간을 요청하면 tablespace가 extent 단위로 공간을 제공
    * 페이지를 하나씩 할당하면 디스크 단편화 발생
## Segment
* InnoDB 내부 객체 중 인덱스가 사용하는 공간 단위
    * *공간을 직접 소유하는 단위는 테이블이 아닌 인덱스*
    * 테이블 = clustered index 1개 + secondary index n개
    * 인덱스 1개 = segment 1개
* 1개의 segment는 여러 개의 extent를 할당 받음
    * 각 extent는 여러 개의 page의 묶음
## Tablespace
* InnoDB가 데이터를 저장하는 물리 파일 단위
* 종류
    * System Tablespace
        * ibdata1
        * 데이터 딕셔너리
        * 일부 메타 데이터
        * `innodb_file_per_table=OFF`일 경우 사용자 테이블도 저장됨
    * Undo Tablespace
        * undo log
    * File-per-Table Tablespace
        * `.ibd` 파일
        * 테이블 1개 당 1개의 `.ibd` 파일 할당
        * 해당 테이블의 모든 인덱스 세그먼트 포함
        * `innodb_file_per_table=ON` (default)
    * General Tablespace
        * 여러 테이블을 하나의 테이블스페이스에 저장 가능
        * `create tablespace`로 직접 생성