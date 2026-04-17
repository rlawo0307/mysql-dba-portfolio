# 목차
* 설정 파일 위치 및 내용 확인
* 설정 변수 확인
<br><br>

# 설정 파일 위치 및 내용 확인
### 설정 파일 경로 확인
```bash
mysqld --help | grep -A 1 "Default options"
mysqld --verbose --help | grep -A 5 "Default options"
```
```bash
Default options are read from the following files in the given order:
/etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf 
```
* `/etc/my.cnf` → `/etc/mysql/my.cnf` → `~/.my.cnf` 순서대로 읽음
* 여러 파일이 있으면 뒤에 있는 파일이 앞 설정을 덮어씀
    * 같은 옵션일 때만 override됨
    * 서로 다른 옵션은 누적 적용됨
### 설정 파일이 존재하는지 확인
* MySQL은 여러 설정 파일을 순차적으로 탐색
* 모든 설정 파일이 존재할 필요는 없음
* 실제 환경에서는 여러 설정 파일 중 일부만 존재하는 경우가 일반적
* 파일이 없다면 무시하고 그냥 넘어감
```bash
for f in /etc/my.cnf /etc/mysql/my.cnf ~/.my.cnf
do
    if [ -f "$f" ]; then
        echo "[O] $f"
    else
        echo "[X] $f"
    fi
done
```
```bash
[X] /etc/my.cnf
[O] /etc/mysql/my.cnf
[X] /home/dbadmin/.my.cnf
```
* 현재 `/etc/mysql/my.cnf` 파일만 존재
### 설정 파일 내용 확인
* `/etc/mysql/my.cnf`는 직접 설정을 담고 있지 않음
    * 다른 설정 파일들을 include 해서 읽게 만드는 파일
    * include된 파일은 `my.cnf`보다 나중에 읽힘
        * `my.cnf` → `conf.d` → `mysql.conf.d`
        * include된 디렉토리는 선언된 순서대로 읽힘
        * 뒤에 있는 파일이 override 가능
```bash
cat /etc/mysql/my.cnf
```
```bash
#
# The MySQL database server configuration file.
#
# You can copy this to one of:
# - "/etc/mysql/my.cnf" to set global options,
# - "~/.my.cnf" to set user-specific options.
# 
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

#
# * IMPORTANT: Additional settings that can override those from this file!
#   The files must end with '.cnf', otherwise they'll be ignored.
#

!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
```
* `/etc/mysql/conf.d/`
    * 추가 설정을 담은 파일
    * 해당 디렉토리 하위의 *.cnf 파일 전부를 읽음
* `/etc/mysql/mysql.conf.d/`
    * 핵심 설정을 담은 파일
    * 해당 디렉토리 하위의 *.cnf 파일 전부를 읽음
* 디렉토리 내 파일은 파일명 기준 알파벳 순으로 읽힘
<br>

# 설정 변수 확인
* MySQL 서버에 현재 어떤 설정 값이 적용되어 있는지 확인해보자
* 실제 적용된 설정 값은 설정 파일이 아닌 `show variables` 결과로 확인해야 함
    * 동적으로 변경된 값이 있을 수 있기 때문에
### 전체 변수 확인
```sql
show variables;
```
```sql
+-------------------------------------+
| Variable_name               | Value |
+-------------------------------------+
| activate_all_roles_on_login | OFF   |
| admin_address               |       |
| admin_port                  | 33062 |
| admin_ssl_ca                |       |
| admin_ssl_capath            |       |
...
```
### 특정 변수 확인
```sql
show variables like 'autocommit';
```
```sql
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| autocommit    | ON    |
+---------------+-------+
1 row in set (0.00 sec)
```
### 동적 변경 가능한 설정 확인
* `performance_schema.variables_info`에서 조회 가능
    * `variable_scope` : 변수 적용 범위 (global/session)
    * `variable_type` : 런타임에 변경 가능 여부(dynamic/static)
    * 단, 모든 MySQL 버전에서 해당 컬럼이 제공되는 것은 아님
* 실제 set 명령으로 변경 가능 여부를 확인하는 것이 가장 확실함
### 변수 설정 변경하기
* 세션 단위 변경
```sql
set session sql_mode = 'STRICT_TRANS_TABLES';
```
* 글로벌 단위 변경
```sql
set global autocommit = 0;
```
* 변경 실패
```sql
set session innodb_log_file_size = 1G; -- fail
set global innodb_log_file_size = 1G; -- fail
```
```sql
ERROR 1238 (HY000): Variable 'innodb_log_file_size' is a read only variable
-- static 변수는 read only 변수로,
-- 변경하려면 설정 파일 수정 후 서버 재시작이 필요
-- 실행 중 변경할 수 없음
```
<br>