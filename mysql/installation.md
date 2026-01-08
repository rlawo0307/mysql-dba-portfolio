# MySQL 설치
```bash
sudo apt update
sudo apt install mysql-server
```
# MySQL 서비스 상태 확인
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
# MySQL에 root 사용자로 접속 시도해보기
```bash
sudo mysql -u root
```
```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 8.0.44-0ubuntu0.24.04.2 (Ubuntu)

Copyright (c) 2000, 2025, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

# root 계정으로 데이터베이스 목록 조회해보기
```sql
show databases;
```
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.01 sec)
```
