# MySQL 설치 및 접속
## MySQL 설치
```bash
sudo apt update
sudo apt install mysql-server
```
## MySQL 서비스 상태 확인
```bash
sudo systemctl status mysql
```
```
[sudo] password for dbadmin: 
● mysql.service - MySQL Community Server
     Loaded: loaded (/usr/lib/systemd/system/mysql.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-01-08 18:53:29 UTC; 4min 17s ago
    Process: 1607 ExecStartPre=/usr/share/mysql/mysql-systemd-start pre (code=exited, >
   Main PID: 1635 (mysqld)
     Status: "Server is operational"
      Tasks: 37 (limit: 4538)
     Memory: 422.3M (peak: 435.0M)
        CPU: 2.169s
     CGroup: /system.slice/mysql.service
             └─1635 /usr/sbin/mysqld

Jan 08 18:53:28 mysql-db systemd[1]: Starting mysql.service - MySQL Community Server...
Jan 08 18:53:29 mysql-db systemd[1]: Started mysql.service - MySQL Community Server.
```
* `Active`
   * 현재 서비스 상태
   * 가능한 값
      * `active (running)` : 정상 실행 중
      * `active (exited)` : 작업 완료 후 종료
      * `inactive (dead)` : 실행 중 아님
      * `failed` : 비정상 종료
      * `activating` : 시작 중
      * `deactivating` : 종료 중
* `Main PID`
   * MySQL Server 프로세스(mysqld)의 PID
* `Status`
   * MySQL 내부 상태
   * `"Server is operational"` 이면 정상적으로 요청을 처리 가능한 상태
## MySQL 접속
* `-u` : 사용자명 지정
* `-p` : 비밀번호 입력
* `-h` : 접속할 host 지정
* `-P` : 포트 번호 지정
```bash
sudo mysql -u root
mysql -u 사용자명 -p
mysql -u 사용자명 -p -h host/IP
mysql -u 사용자명 -p -h host/IP -P 포트번호
```