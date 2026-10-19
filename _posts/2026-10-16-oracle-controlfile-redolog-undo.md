---
title: "Oracle 19c Control File 및 Redo Log 이관과 Undo 및 사용자 권한 실습 | 30일차"
date: 2026-10-16 00:00:00 +0900
categories: [Oracle]
tags: [Oracle 19c, Control File, Redo Log, Undo]
---

---


#control file 이관
```plain text
/u01/app/oracle/oradata/ORA19C/control01.ctl -> /home/oracle/userdata/control01.ctl
```

1. **초기파라미터 파일이 pfile, spfile 체크**
```bash
SYS@ora19c> show parameter spfile

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      /u01/app/oracle/product/19.3.0
                                                 /dbhome_1/dbs/spfileora19c.ora
```

1. control_files 파라미터값을 수정
```bash
SYS@ora19c> show parameter control_files

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string      /u01/app/oracle/oradata/ORA19C
                                                 /control01.ctl
                                                 
SYS@ora19c> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/control01.ctl


alter system set control_files = '/home/oracle/userdata/control01.ctl' scope = spfile;


```

```bash
SYS@ora19c> select member from v$logfile;


MEMBER
--------------------------------------------------------------------------------
/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo02.log
/u01/app/oracle/oradata/ORA19C/redo3.log
/u01/app/oracle/oradata/ORA19C/redo06.log

SELECT * FROM v$log;
```

#redo logfile 이관
```plain text
/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo02.log -> /home/oracle/userdata/redo02.log
/u01/app/oracle/oradata/ORA19C/redo3.log -> /home/oracle/userdata/redo3.log
/u01/app/oracle/oradata/ORA19C/redo06.log -> /home/oracle/userdata/redo06.log
```

- 데이터베이스에 대한 정보, 데이터 파일도, redo log도  controlfile이 가지고 있고 있다

```bash
alter database rename file '이전 주소' to '새로운 주소';

alter database rename file '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo02.log' 
to '/home/oracle/userdata/redo02.log';
alter database rename file '/u01/app/oracle/oradata/ORA19C/redo3.log' 
to '/home/oracle/userdata/redo3.log';
alter database rename file '/u01/app/oracle/oradata/ORA19C/redo06.log' 
to '/home/oracle/userdata/redo06.log';
```

#control file, redo log file 이관 실습
1. **초기파라미터 파일이 pfile, spfile 체크**
```bash
SYS@ora19c> show parameter spfile

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
spfile                               string      /u01/app/oracle/product/19.3.0
                                                 /dbhome_1/dbs/spfileora19c.ora
```

1. control_files 파라미터값을 수정
```bash
SYS@ora19c> show parameter control_files

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
control_files                        string      /u01/app/oracle/oradata/ORA19C
                                                 /control01.ctl
                                                 
SYS@ora19c> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/control01.ctl


#alter system set control_files = '/home/oracle/userdata/control01.ctl' scope = spfile;


SYS@ora19c> alter system set control_files = '/home/oracle/userdata/control01.ctl' scope = spfile;


System altered.

SYS@ora19c> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/control01.ctl

```

1. 데이터베이스 정상적인 종료
```bash
SYS@ora19c> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.
SYS@ora19c> !
```

1. 기존 컨트롤파일, 리두로그 파일로 새로운 위치로 이동
```bash
mv -v /u01/app/oracle/oradata/ORA19C/control01.ctl /home/oracle/userdata/
mv -v /u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo02.log /home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/redo3.log /home/oracle/userdata/
mv -v /u01/app/oracle/oradata/ORA19C/redo06.log /home/oracle/userdata/

[oracle@oracle19c ~]$ ls ~/userdata/
control01.ctl  redo06.log  sysaux01.dbf  temp01.dbf     userdata01.dbf
redo02.log     redo3.log   system01.dbf  undotbs01.dbf  users01.dbf

```

1. 데이터베이스를 mount 시작
```bash
SYS@ora19c> startup mount
ORACLE instance started.

Total System Global Area 1241513488 bytes
Fixed Size                  8896016 bytes
Variable Size             318767104 bytes
Database Buffers          905969664 bytes
Redo Buffers                7880704 bytes
Database mounted.
SYS@ora19c> select instance_name, status from v$instance;

INSTANCE_NAME    STATUS
---------------- ------------
ora19c           MOUNTED

SYS@ora19c> select name from v$controlfile;

NAME
--------------------------------------------------------------------------------
/home/oracle/userdata/control01.ctl


#지금 컨트롤 파일안에는 리두로그 위치를 이전 위치를 가지고 있다. 새로운 위치로 변경해야한다.
SYS@ora19c>  select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo02.log
/u01/app/oracle/oradata/ORA19C/redo3.log
/u01/app/oracle/oradata/ORA19C/redo06.log

alter database rename file '/u01/app/oracle/fast_recovery_area/ORA19C/onlinelog/redo02.log' 
to '/home/oracle/userdata/redo02.log';
alter database rename file '/u01/app/oracle/oradata/ORA19C/redo3.log' 
to  '/home/oracle/userdata/redo3.log';
alter database rename file '/u01/app/oracle/oradata/ORA19C/redo06.log' 
to '/home/oracle/userdata/redo06.log';

SYS@ora19c> select member from v$logfile;

MEMBER
--------------------------------------------------------------------------------
/home/oracle/userdata/redo02.log
/home/oracle/userdata/redo3.log
/home/oracle/userdata/redo06.log

SYS@ora19c> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/home/oracle/userdata/system01.dbf
/home/oracle/userdata/sysaux01.dbf
/home/oracle/userdata/undotbs01.dbf
/home/oracle/userdata/userdata01.dbf
/home/oracle/userdata/users01.dbf


```
6.?

1. 데이터베이스 시작
```bash
SYS@ora19c> alter database open;
Database altered.

```

\[문제\]  모든 컨트롤파일, 리두로그파일, 데이터파일 이관
```bash
SYS@ora19c> ! ls -lh /home/oracle/userdata/*.{ctl,log,dbf}
-rw-r-----. 1 oracle oinstall  11M Oct 15 22:19 /home/oracle/userdata/control01.ctl
-rw-r-----. 1 oracle oinstall  51M Oct 15 22:19 /home/oracle/userdata/redo02.log
-rw-r-----. 1 oracle oinstall  51M Oct 15 22:16 /home/oracle/userdata/redo06.log
-rw-r-----. 1 oracle oinstall  51M Oct 15 22:16 /home/oracle/userdata/redo3.log
-rw-r-----. 1 oracle oinstall 661M Oct 15 22:17 /home/oracle/userdata/sysaux01.dbf
-rw-r-----. 1 oracle oinstall 911M Oct 15 22:16 /home/oracle/userdata/system01.dbf
-rw-r-----. 1 oracle oinstall 131M Oct 15 22:00 /home/oracle/userdata/temp01.dbf
-rw-r-----. 1 oracle oinstall 346M Oct 15 22:16 /home/oracle/userdata/undotbs01.dbf
-rw-r-----. 1 oracle oinstall 5.1M Oct 15 22:16 /home/oracle/userdata/userdata01.dbf
-rw-r-----. 1 oracle oinstall 5.1M Oct 15 22:16 /home/oracle/userdata/users01.dbf

SYS@ora19c> ! ls -lh /u01/app/oracle/oradata/ORA19C/
total 0

/home/oracle/userdata/*.{ctl,log,dbf} -> /u01/app/oracle/oradata/ORA19C/
```

1. **파일 전부 다 옮기는 법**
```bash

alter system set control_files = '/u01/app/oracle/oradata/ORA19C/control01.ctl' scope = spfile;

SHUTDOWN IMMEDIATE;

! mv -v /home/oracle/userdata/*.{ctl,log,dbf} /u01/app/oracle/oradata/ORA19C/

startup mount 

select name from v$controlfile;

-- 현재 Oracle에 등록된 Data File 경로 확인
SELECT name FROM v$datafile;

-- Data File의 파일명을 추출하여 경로 변경 SQL 자동 생성
-- /home/oracle/userdata/ → /u01/app/oracle/oradata/ORA19C/

SELECT 'ALTER DATABASE RENAME FILE ''' || name ||
       ''' TO ''/u01/app/oracle/oradata/ORA19C/' ||
       SUBSTR(name, INSTR(name, '/', -1, 1) + 1) ||
       ''';'
FROM v$datafile;

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/system01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/system01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/sysaux01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/sysaux01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/undotbs01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/users01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/users01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/userdata01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/userdata01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/redo02.log'
TO '/u01/app/oracle/oradata/ORA19C/redo02.log';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/redo3.log'
TO '/u01/app/oracle/oradata/ORA19C/redo3.log';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/redo06.log'
TO '/u01/app/oracle/oradata/ORA19C/redo06.log';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/temp01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/temp01.dbf';

SELECT name FROM v$datafile;
SELECT name FROM v$tempfile;
SELECT member FROM v$logfile;
SELECT name FROM v$controlfile;

alter database open;

! ls -lh /home/oracle/userdata/*.{ctl,log,dbf}
```


![](/assets/img/oracle-2026-10-16/01.png)
- 일반 Tablespace 하나: `OFFLINE NORMAL → mv → ALTER TABLESPACE RENAME DATAFILE → ONLINE`
- SYSTEM을 포함한 전체 파일: `SHUTDOWN → mv → STARTUP MOUNT → ALTER DATABASE RENAME FILE → OPEN`

- NOMOUNT: 인스턴스만 시작하므로 SPFILE 설정을 수정할 수 있음
- MOUNT: Control File을 열었으므로 Data File과 Redo Log File의 등록 경로를 수정할 수 있음
- OPEN: Data File까지 열어 일반적인 SQL 작업을 수행할 수 있음

**#내가 직접 수행한거**
```bash

 SYS@ora19c> alter system set control_files = '/u01/app/oracle/oradata/ORA19C/' scope = spfile;
--control01.ctl 빼먹었
System altered.

SYS@ora19c> SHUTDOWN IMMEDIATE;
Database closed.
Database dismounted.
^CORA-12152: TNS:unable to send break message
-- 안 켜진다 컨트롤 파일 정보가 잘못되서

SYS@ora19c> STARTUP NOMOUNT;
SP2-0642: SQL*Plus internal error state 2133, context 3114:0:0
Unsafe to proceed
ORA-03114: not connected to ORACLE


SYS@ora19c> !
[oracle@oracle19c ~]$ sqlplus / as sysdba

SQL*Plus: Release 19.0.0.0.0 - Production on Thu Oct 15 22:57:54 2026
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

Connected to an idle instance.

SYS@ora19c> STARTUP NOMOUNT;
ORACLE instance started.

Total System Global Area 1241513488 bytes
Fixed Size                  8896016 bytes
Variable Size             318767104 bytes
Database Buffers          905969664 bytes
Redo Buffers                7880704 bytes

SYS@ora19c> ALTER SYSTEM SET control_files =
'/u01/app/oracle/oradata/ORA19C/control01.ctl'
SCOPE=SPFILE;  2    3

System altered.

SYS@ora19c> SHUTDOWN IMMEDIATE;
ORA-01507: database not mounted

ORACLE instance shut down.

SYS@ora19c> ! ls -lh /home/oracle/userdata/control01.ctl
! ls -lh /u01/app/oracle/oradata/ORA19C/control01.ctl-rw-r-----. 1 oracle oinstall 11M Oct 15 22:55 /home/oracle/userdata/control01.ctl


SYS@ora19c> ! ls -lh /home/oracle/userdata/control01.ctl
-rw-r-----. 1 oracle oinstall 11M Oct 15 22:55 /home/oracle/userdata/control01.ctl

SYS@ora19c> ! ls -lh /u01/app/oracle/oradata/ORA19C/control01.ctl
ls: cannot access /u01/app/oracle/oradata/ORA19C/control01.ctl: No such file or directory

SYS@ora19c> ! mv -iv /home/oracle/userdata/*.{ctl,log,dbf} /u01/app/oracle/oradata/ORA19C/
‘/home/oracle/userdata/control01.ctl’ -> ‘/u01/app/oracle/oradata/ORA19C/control01.ctl’
removed ‘/home/oracle/userdata/control01.ctl’
‘/home/oracle/userdata/redo02.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo02.log’
removed ‘/home/oracle/userdata/redo02.log’
‘/home/oracle/userdata/redo06.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo06.log’
removed ‘/home/oracle/userdata/redo06.log’
‘/home/oracle/userdata/redo3.log’ -> ‘/u01/app/oracle/oradata/ORA19C/redo3.log’
removed ‘/home/oracle/userdata/redo3.log’
‘/home/oracle/userdata/sysaux01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/sysaux01.dbf’
removed ‘/home/oracle/userdata/sysaux01.dbf’
‘/home/oracle/userdata/system01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
removed ‘/home/oracle/userdata/system01.dbf’
‘/home/oracle/userdata/temp01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/temp01.dbf’
removed ‘/home/oracle/userdata/temp01.dbf’
‘/home/oracle/userdata/undotbs01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
removed ‘/home/oracle/userdata/undotbs01.dbf’
‘/home/oracle/userdata/userdata01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/userdata01.dbf’
removed ‘/home/oracle/userdata/userdata01.dbf’
‘/home/oracle/userdata/users01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/users01.dbf’
removed ‘/home/oracle/userdata/users01.dbf’

SYS@ora19c> STARTUP MOUNT;
ORACLE instance started.

Total System Global Area 1241513488 bytes
Fixed Size                  8896016 bytes
Variable Size             318767104 bytes
Database Buffers          905969664 bytes
Redo Buffers                7880704 bytes
Database mounted.
SYS@ora19c> SELECT name FROM v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/control01.ctl

SYS@ora19c> ALTER DATABASE RENAME FILE
'/home/oracle/userdata/system01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/system01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/sysaux01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/sysaux01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/undotbs01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/users01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/users01.dbf';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/userdata01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/userdata01.dbf';  
Database altered.

SYS@ora19c> 
Database altered.

SYS@ora19c>
Database altered.

SYS@ora19c> 
Database altered.

SYS@ora19c> 

Database altered.

SYS@ora19c> ALTER DATABASE RENAME FILE
'/home/oracle/userdata/temp01.dbf'
TO '/u01/app/oracle/oradata/ORA19C/temp01.dbf';

Database altered.

SYS@ora19c> ALTER DATABASE RENAME FILE
'/home/oracle/userdata/redo02.log'
TO '/u01/app/oracle/oradata/ORA19C/redo02.log';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/redo3.log'
TO '/u01/app/oracle/oradata/ORA19C/redo3.log';

ALTER DATABASE RENAME FILE
'/home/oracle/userdata/redo06.log'
TO '/u01/app/oracle/oradata/ORA19C/redo06.log';
Database altered.

SYS@ora19c> 
Database altered.

SYS@ora19c> 

Database altered.

SYS@ora19c> SELECT name FROM v$controlfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/control01.ctl

SYS@ora19c> SELECT name FROM v$datafile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/system01.dbf
/u01/app/oracle/oradata/ORA19C/sysaux01.dbf
/u01/app/oracle/oradata/ORA19C/undotbs01.dbf
/u01/app/oracle/oradata/ORA19C/userdata01.dbf
/u01/app/oracle/oradata/ORA19C/users01.dbf

SYS@ora19c> SELECT name FROM v$tempfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/temp01.dbf

SYS@ora19c> SELECT member FROM v$logfile;

MEMBER
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/redo02.log
/u01/app/oracle/oradata/ORA19C/redo3.log
/u01/app/oracle/oradata/ORA19C/redo06.log

SYS@ora19c> ALTER DATABASE OPEN;

Database altered.

SYS@ora19c> SELECT name, open_mode FROM v$database;

SELECT tablespace_name, status
FROM dba_tablespaces;

SELECT COUNT(*) FROM hr.emp;
NAME      OPEN_MODE
--------- --------------------
ORA19C    READ WRITE

SYS@ora19c> SYS@ora19c>   2
TABLESPACE_NAME                STATUS
------------------------------ ---------
SYSTEM                         ONLINE
SYSAUX                         ONLINE
UNDOTBS1                       ONLINE
TEMP                           ONLINE
USERS                          ONLINE
USERDATA                       ONLINE

6 rows selected.

SYS@ora19c> SYS@ora19c>

  COUNT(*)
----------
       107

SYS@ora19c> SELECT tablespace_name, status
FROM dba_tablespaces;  2

TABLESPACE_NAME                STATUS
------------------------------ ---------
SYSTEM                         ONLINE
SYSAUX                         ONLINE
UNDOTBS1                       ONLINE
TEMP                           ONLINE
USERS                          ONLINE
USERDATA                       ONLINE

6 rows selected.

SYS@ora19c> SELECT COUNT(*) FROM hr.emp;

  COUNT(*)
----------
       107

SYS@ora19c> ! ls -lh /u01/app/oracle/oradata/ORA19C/
total 2.2G
-rw-r-----. 1 oracle oinstall  11M Oct 15 23:00 control01.ctl
-rw-r-----. 1 oracle oinstall  51M Oct 15 23:00 redo02.log
-rw-r-----. 1 oracle oinstall  51M Oct 15 23:00 redo06.log
-rw-r-----. 1 oracle oinstall  51M Oct 15 23:00 redo3.log
-rw-r-----. 1 oracle oinstall 671M Oct 15 23:00 sysaux01.dbf
-rw-r-----. 1 oracle oinstall 911M Oct 15 23:00 system01.dbf
-rw-r-----. 1 oracle oinstall 131M Oct 15 22:00 temp01.dbf
-rw-r-----. 1 oracle oinstall 346M Oct 15 23:00 undotbs01.dbf
-rw-r-----. 1 oracle oinstall 5.1M Oct 15 23:00 userdata01.dbf
-rw-r-----. 1 oracle oinstall 5.1M Oct 15 23:00 users01.dbf

SYS@ora19c> ! ls -lh /home/oracle/userdata/*.{ctl,log,dbf}
ls: cannot access /home/oracle/userdata/*.ctl: No such file or directory
ls: cannot access /home/oracle/userdata/*.log: No such file or directory
ls: cannot access /home/oracle/userdata/*.dbf: No such file or directory

```

■ undo
- DML 작업시 이전값을 저장하는 공간
- 트랜잭션이 종료될때 까지는 이전값을 보존해야한다.
- 목적
	- rollback
	- read consistency(읽기 일관성)
	- instance failure가 발생한 경우 실패한 트랜잭션 recovery시 필요하다.
	- flashback query(10g)
- undo monitoring
	- undo space가 부족한 경우 ORA-01650
	- 읽기 일관성이 어긋난 경우 ORA-01555
	- enq : US - contention 대기 이벤트
-
```bash
SYS@ora19c> show parameter undo_tablespace

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_tablespace                      string      UNDOTBS1
SYS@ora19c> show parameter undo_management

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_management                      string      AUTO <<- undo segment관리 자동으로 수행

#undo_retention
- 트랜잭션이 종료가 되었더라도 이전값을 undo_retention파라미터 설정되어 있는 초까지 보존하겠다.(옵션)
- 읽기 일관성, flashback query


SYS@ora19c> show parameter undo_retention

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_retention                       integer     900 <<- 초단위

select * from v$parameter
where name in ('undo_tablespace', 'undo_management', 'undo_retention');

```
![](/assets/img/oracle-2026-10-16/02.png)
- 시스템 테이블 스페이스는 딕셔너리 테이블 정보가 저장되어 있다. 
- 딕셔너리 정보는 ddl 문장이 수행될때 수정된다. 
- 이때 dml 문장이 수행되는데 undo가 만들어진다

```bash
SYS@ora19c> SELECT n.usn, n.name, s.extents, s.rssize, s.xacts, s.status
             FROM v$rollname n, v$rollstat s
             WHERE n.usn = s.usn;

       USN NAME                         EXTENTS     RSSIZE      XACTS STATUS
---------- ------------------------- ---------- ---------- ---------- ----------
         0 SYSTEM                             7     450560          0 ONLINE
         1 _SYSSMU1_1261223759$               5    3268608          0 ONLINE
         2 _SYSSMU2_27624015$                 6    4317184          0 ONLINE
         3 _SYSSMU3_2421748942$               4    2220032          0 ONLINE
         4 _SYSSMU4_625702278$                4    2220032          0 ONLINE
         5 _SYSSMU5_2101348960$               6    4317184          0 ONLINE
         6 _SYSSMU6_813816332$                5    3268608          0 ONLINE
         7 _SYSSMU7_2329891355$               5    3268608          0 ONLINE
         8 _SYSSMU8_399776867$                6    3334144          0 ONLINE
         9 _SYSSMU9_1692468413$               5    3268608          0 ONLINE
        10 _SYSSMU10_930580995$               6    4317184          0 ONLINE

11 rows selected.


HR@ora19c> select salary from hr.employees where employee_id = 200;

    SALARY
----------
      4400

<<session 1>>

HR@ora19c> update hr.employees set salary = 5000 where employee_id = 200;

1 row updated.

HR@ora19c> select salary from hr.employees where employee_id = 200;

    SALARY
----------
      5000
```
- 0번은 딕셔너리 정보에 대한 undo이다
- XACTS : 트랜잭션이 몇개나 붙어 있나 표시
- STATUS : db켤때 알아서 켜진다

- 오라클은 undo랑 CR이 있어서 트랜잭션하고 있는 데이터도 다른 사람도 볼 수 있다.
```bash
SYS@ora19c> SELECT s.username, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
             FROM v$session s, v$transaction t
             WHERE s.saddr = t.ses_addr;

USERNAME           XIDUSN     UBAFIL     UBABLK  USED_UBLK
--------------- ---------- ---------- ---------- ----------
HR                       8          4       2503          1
```
- XIDUSN : UNDO 세그먼트 id
- ubafil : undo 세그먼트가 속해져 있는 파일번호
- undo 헤더블락에 트랜잭션 정보를 담아 놓는다

```bash
select * from v$session;

-- 현재 활성 트랜잭션 상세 정보 확인
SELECT * FROM v$transaction;
```
- 이걸로 세션 모니터링 한다

#같은 사용자에서 다른 세션으로 들어올때
```bash
<<session 2>>
HR@ora19c> update hr.employees set salary = salary * 1.5 where employee_id = 150;

1 row updated.

select salary from hr.employees where employee_id = 150;

    SALARY
----------
     15000

SYS@ora19c> SELECT n.usn, n.name, s.extents, s.rssize, s.xacts, s.status
             FROM v$rollname n, v$rollstat s
             WHERE n.usn = s.usn;

       USN NAME                         EXTENTS     RSSIZE      XACTS STATUS
---------- ------------------------- ---------- ---------- ---------- ----------
         0 SYSTEM                             7     450560          0 ONLINE
         1 _SYSSMU1_1261223759$               5    3268608          0 ONLINE
         2 _SYSSMU2_27624015$                 6    4317184          0 ONLINE
         3 _SYSSMU3_2421748942$               4    2220032          0 ONLINE
         4 _SYSSMU4_625702278$                4    2220032          0 ONLINE
         5 _SYSSMU5_2101348960$               6    4317184          1 ONLINE
         6 _SYSSMU6_813816332$                5    3268608          0 ONLINE
         7 _SYSSMU7_2329891355$               5    3268608          0 ONLINE
         8 _SYSSMU8_399776867$                6    3334144          1 ONLINE
         9 _SYSSMU9_1692468413$               4    2220032          0 ONLINE
        10 _SYSSMU10_930580995$               6    4317184          0 ONLINE

11 rows selected.


SYS@ora19c> SELECT s.username, s.sid, s.serial#, t.xidusn,
                   t.ubafil, t.ubablk, t.used_ublk
             FROM v$session s, v$transaction t
             WHERE s.saddr = t.ses_addr;

USERNAME               SID    SERIAL#     XIDUSN     UBAFIL     UBABLK  USED_UBLK
--------------- ---------- ---------- ---------- ---------- ---------- ----------
HR                     406      29909          5          4       4451          1
HR                      38       6823          8          4       2503
```
- 오라클은 하나의 세션에 하나의 undo를 줄려고 하고 만약에 부족하면 10개를 추가적으로 만든다
- 근데 만약에 undo 공간을 더 만들수 없을때는 undo를 같이 쓴다. 그래도 부족하면 오류가 발생한다.

**#session kill**
```bash
alter system kill session '38,6823' immediate; #세션 1kill

SYS@ora19c> SELECT s.username, s.sid, s.serial#, t.xidusn,
                   t.ubafil, t.ubablk, t.used_ublk
             FROM v$session s, v$transaction t
             WHERE s.saddr = t.ses_addr;

USERNAME               SID    SERIAL#     XIDUSN     UBAFIL     UBABLK  USED_UBLK
--------------- ---------- ---------- ---------- ---------- ---------- ----------
HR                     406      29909          5          4       4451          1


<<session1>>

HR@ora19c> select salary from hr.employees where employee_id = 200;
select salary from hr.employees where employee_id = 200
*
ERROR at line 1:
ORA-12571: TNS:packet writer failure


HR@ora19c> ^C

HR@ora19c> update hr.employees set salary = 5000 where employee_id = 200;
ERROR:
ORA-03114: not connected to ORACLE


```
- kill 시키면 pmon이 rollback해준

#undo tablespace에 undo data file 추가
```bash
ALTER TABLESPACE undotbs1 ADD DATAFILE
'/u01/app/oracle/oradata/ORA19C/undotbs02.dbf'
SIZE 10M AUTOEXTEND ON;

select * from dba_data_files;
```
![2개로 늘어남](/assets/img/oracle-2026-10-16/03.png)
```bash
SYS@ora19c> ! ls -lh /u01/app/oracle/oradata/ORA19C/undotbs02.dbf
-rw-r-----. 1 oracle oinstall 11M Oct 16 02:04 /u01/app/oracle/oradata/ORA19C/undotbs02.dbf

```
#Undo 파일을 옮기고 싶은데 offline으로 내리지는 못한다.
#새로운 undo tablespace 생성
```bash
create undo tablespace undo1 
datafile '/u01/app/oracle/oradata/ORA19C/undo01.dbf' size 10m autoextend on;

select * from dba_tablespaces;
select * from dba_data_files;
```
![](/assets/img/oracle-2026-10-16/04.png)
![](/assets/img/oracle-2026-10-16/05.png)
- 기본 적인 db에서는 undo를 하나만 쓰기 때문에 하나를 추가해도 그냥 기존에 있던 하나만 사용한다
- rac만 노드 하나당 undo 테이블 하나를 사용한다.  그래서 노드가 여러개만 undo테이블도 여러개일 수 있다. 
```bash
SYS@ora19c> SHOW PARAMETER undo_tablespace;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_tablespace                      string      UNDOTBS1



<세션 1>
HR@ora19c> update hr.employees set salary = salary * 1.5 where employee_id = 150;


SYS@ora19c> SELECT s.username, s.sid, s.serial#, t.xidusn,
                   t.ubafil, t.ubablk, t.used_ublk
             FROM v$session s, v$transaction t
             WHERE s.saddr = t.ses_addr;

USERNAME               SID    SERIAL#     XIDUSN     UBAFIL     UBABLK  USED_UBLK
--------------- ---------- ---------- ---------- ---------- ---------- ----------
HR                     406      29909          8          4       2506



SYS@ora19c> SELECT n.usn, n.name, s.extents, s.rssize, s.xacts, s.status
             FROM v$rollname n, v$rollstat s
             WHERE n.usn = s.usn;

       USN NAME                         EXTENTS     RSSIZE      XACTS STATUS
---------- ------------------------- ---------- ---------- ---------- ----------
         0 SYSTEM                             7     450560          0 ONLINE
         1 _SYSSMU1_1261223759$               5    3268608          0 ONLINE
         2 _SYSSMU2_27624015$                 6    4317184          0 ONLINE
         3 _SYSSMU3_2421748942$               4    2220032          0 ONLINE
         4 _SYSSMU4_625702278$                4    2220032          0 ONLINE
         5 _SYSSMU5_2101348960$               6    4317184          0 ONLINE
         6 _SYSSMU6_813816332$                5    3268608          0 ONLINE
         7 _SYSSMU7_2329891355$               5    3268608          0 ONLINE
         8 _SYSSMU8_399776867$                6    3334144          1 ONLINE
         9 _SYSSMU9_1692468413$               4    2220032          0 ONLINE
        10 _SYSSMU10_930580995$               6    4317184          0 ONLINE

11 rows selected.
```

- undo  관리만 static , undo_tablespace는 다이나믹 파라미터
#새로운 undo tablespace 지정
```bash
SYS@ora19c> SHOW PARAMETER undo_tablespace;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_tablespace                      string      UNDOTBS1

alter system set undo_tablespace = undo1 scope = both



SYS@ora19c> alter system set undo_tablespace = undo1 scope = both;

System altered.

SYS@ora19c> SHOW PARAMETER undo_tablespace;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_tablespace                      string      UNDO1

SYS@ora19c> SELECT n.usn, n.name, s.extents, s.rssize, s.xacts, s.status
             FROM v$rollname n, v$rollstat s
             WHERE n.usn = s.usn;

       USN NAME                         EXTENTS     RSSIZE      XACTS STATUS
---------- ------------------------- ---------- ---------- ---------- ---------------
         0 SYSTEM                             7     450560          0 ONLINE
         8 _SYSSMU8_399776867$                6    3334144          1 PENDING OFFLINE
        11 _SYSSMU11_951741226$               2     122880          0 ONLINE
        12 _SYSSMU12_62613116$                2     122880          0 ONLINE
        13 _SYSSMU13_1858846385$              2     122880          0 ONLINE
        14 _SYSSMU14_511009641$               2     122880          0 ONLINE
        15 _SYSSMU15_1185041822$              2     122880          0 ONLINE
        16 _SYSSMU16_4173398250$              2     122880          0 ONLINE
        17 _SYSSMU17_3775494503$              2     122880          0 ONLINE
        18 _SYSSMU18_3496659576$              2     122880          0 ONLINE
        19 _SYSSMU19_3280121143$              2     122880          0 ONLINE
        20 _SYSSMU20_3494489525$              2     122880          0 ONLINE

12 rows selected.


<<session 2>>
HR@ora19c> update hr.employees set salary = 5000 where employee_id = 200;

1 row updated.

<< SYS SESSION >>

SYS@ora19c> SELECT n.usn, n.name, s.extents, s.rssize, s.xacts, s.status
             FROM v$rollname n, v$rollstat s
             WHERE n.usn = s.usn;

       USN NAME                         EXTENTS     RSSIZE      XACTS STATUS
---------- ------------------------- ---------- ---------- ---------- ---------------
         0 SYSTEM                             7     450560          0 ONLINE
         8 _SYSSMU8_399776867$                6    3334144          1 PENDING OFFLINE
        11 _SYSSMU11_951741226$               2     122880          0 ONLINE
        12 _SYSSMU12_62613116$                2     122880          0 ONLINE
        13 _SYSSMU13_1858846385$              2     122880          0 ONLINE
        14 _SYSSMU14_511009641$               2     122880          0 ONLINE
        15 _SYSSMU15_1185041822$              2     122880          0 ONLINE
        16 _SYSSMU16_4173398250$              2     122880          0 ONLINE
        17 _SYSSMU17_3775494503$              2     122880          1 ONLINE
        18 _SYSSMU18_3496659576$              2     122880          0 ONLINE
        19 _SYSSMU19_3280121143$              2     122880          0 ONLINE
        20 _SYSSMU20_3494489525$              2     122880          0 ONLINE

12 rows selected.

#PENDING OFFLINE : 기존 트랜잭션이 끝나기 기다리는 중

SYS@ora19c> SELECT s.username, s.sid, s.serial#, t.xidusn,
                   t.ubafil, t.ubablk, t.used_ublk
             FROM v$session s, v$transaction t
             WHERE s.saddr = t.ses_addr;

USERNAME               SID    SERIAL#     XIDUSN     UBAFIL     UBABLK  USED_UBLK
--------------- ---------- ---------- ---------- ---------- ---------- ----------
HR                     426      13365         17          8        226          1
HR                     406      29909          8          4       2506


<<session 1,2>>
HR@ora19c> rollback;

Rollback complete.

SYS@ora19c> SELECT s.username, s.sid, s.serial#, t.xidusn,
                   t.ubafil, t.ubablk, t.used_ublk
             FROM v$session s, v$transaction t
             WHERE s.saddr = t.ses_addr;

no rows selected

SYS@ora19c> SELECT n.usn, n.name, s.extents, s.rssize, s.xacts, s.status
             FROM v$rollname n, v$rollstat s
             WHERE n.usn = s.usn;

       USN NAME                         EXTENTS     RSSIZE      XACTS STATUS
---------- ------------------------- ---------- ---------- ---------- ----------
         0 SYSTEM                             7     450560          0 ONLINE
        11 _SYSSMU11_951741226$               2     122880          0 ONLINE
        12 _SYSSMU12_62613116$                2     122880          0 ONLINE
        13 _SYSSMU13_1858846385$              2     122880          0 ONLINE
        14 _SYSSMU14_511009641$               2     122880          0 ONLINE
        15 _SYSSMU15_1185041822$              2     122880          0 ONLINE
        16 _SYSSMU16_4173398250$              2     122880          0 ONLINE
        17 _SYSSMU17_3775494503$              2     122880          0 ONLINE
        18 _SYSSMU18_3496659576$              2     122880          0 ONLINE
        19 _SYSSMU19_3280121143$              2     122880          0 ONLINE
        20 _SYSSMU20_3494489525$              2     122880          0 ONLINE

11 rows selected.


```
- 기존에 있던 게 트랜잭션이 끝나서 다 내려갔다.

#undo tablespace 삭제
- undo tablespace에 속한 undo segment 전부 offline 상태이기 때문에 삭제가능하다.
- 만약에 PENDING OFFLINE상태가 있을 경우는 삭제가 불가능하다.
```bash
SYS@ora19c> drop tablespace undotbs1 including contents and datafiles;

Tablespace dropped.
# 어떤 트랜잭션도 사용 안하고 있으면 떨어트릴 수 있다.
```

#새로운 undo tablespace 생성
```bash
create undo tablespace undotbs1
datafile '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf' size 10m autoextend on;

alter system set undo_tablespace = undotbs1 scope = both;

SYS@ora19c> show parameter undo_tablespace;

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
undo_tablespace                      string      UNDOTBS1

SYS@ora19c> drop tablespace undo1 including contents and datafiles;

Tablespace dropped.

```

■ 유저 생성
```bash
CREATE USER 유저이름
IDENTIFIED BY 패스워드; -- 이것만 있어도 생성된다. 

CREATE USER 유저이름
IDENTIFIED BY 패스워드
DEFAULT TABLESPACE 테이블스페이스이름
TEMPORARY TABLESPACE 임시테이블스페이스이름
QUOTA unlimited ON 테이블스페이스
QUOTA 10M ON 테이블스페이스
ACCOUNT [UNLOCK(기본값) | LOCK]
PROFILE [프로파일 | DEFAULT(기본값)]
PASSWORD EXPIRE;
```
- 유저이름 : 데이터베이스에서 고유한 이름으로 설정
- 패스워드 : 데이터베이스 로그인시에 사용할 비밀번호
- PASSWORD EXPIRE : 유저가 로그인할때 암호를 재설정하도록하는 기능
- PROFILE : 리소스, 패스워드 관리
- QUOTA : 테이블스페이스를 사용할 수 있는 권한(양), K, M, UNLIMITED
- DEFAULT TABLESPACE
	- 예)
	- create table hr.emp(id number, name varchar2(30)); \<← default tablespace로 지정된 테이블스페이스에 테이블이 생성
	- create table hr.emp(id number, name varchar2(30)) tablespace users;
TEMPORARY TABLESPACE : 정렬, 해시, 비트맵 인덱스 생성, 병합시에 메모리에서 처리를 다 못할 경우 사용하는 디스크 영역
- SQL Work Areas: PGA 내부에서 SQL 실행 시 정렬(Sort) 및 해시(Hash Join)와 비트맵 인덱스 생성 및 병합 등에 사용하는 작업용 메모리 영역

■ 유저 수정
```bash
ALTER USER 유저이름 -- 필수 구문
IDENTIFIED BY 패스워드
DEFAULT TABLESPACE 테이블스페이스이름
TEMPORARY TABLESPACE 임시테이블스페이스이름
QUOTA unlimited ON 테이블스페이스
QUOTA 10M ON 테이블스페이스
ACCOUNT [UNLOCK(기본값) | LOCK]
PROFILE [프로파일 | DEFAULT(기본값)]
PASSWORD EXPIRE;

-- 나머지는 필요한 것들만 쓰면 된다
```

■ 유저 삭제
DROP USER 유저이름 CASCADE;

- CASCADE : 유저가 생성한 객체들을 먼저 삭제한 후 유저 삭제하는 옵션
- 현재 접속한 유저는 삭제할 수 없다.

```bash
ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE;
```
- TEMPORARY TABLESPACE는 유저별로 나눠서 사용한다

```bash
create tablespace insa_tbs
datafile '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf' size 10M autoextend on;

create temporary tablespace insa_temp
tempfile '/u01/app/oracle/oradata/ORA19C/insa_temp01.dbf' size 10M autoextend on;

select * from dba_data_files;
select * from dba_temp_files;
select name from v$datafile;
select name from v$tempfile;

CREATE USER insa
IDENTIFIED BY oracle
DEFAULT TABLESPACE insa_tbs
TEMPORARY TABLESPACE insa_temp
QUOTA unlimited ON insa_tbs
PASSWORD EXPIRE;

select * from dba_users where username = 'INSA';
select * from dba_ts_quotas where username = 'INSA';
```
![](/assets/img/oracle-2026-10-16/06.png)
![](/assets/img/oracle-2026-10-16/07.png)
![](/assets/img/oracle-2026-10-16/08.png)
![](/assets/img/oracle-2026-10-16/09.png)
![](/assets/img/oracle-2026-10-16/10.png)
temporary_tablespace를 먼저 쓰고 그 다음에 로컬을 사용

■  권한(privilege)
- 권한은 특정한 SQL문을 실행하거나 다른 유저가 소유한 객체를 액세스할 수 있는 권한을 의미
#시스템 권한
- 데이터베이스에서 특정한 작업을 수행할 수 있는 권한, 데이터베이스에 영향을 줄 수 있는 권한
- 권한 관리 SYS가 한다

#객체권한
- 다른 유저가 소유한 객체를 액세스할 수 있는 권한을 의미한다.
- 권한관리는 SYS 또는 객체 소유자가 한다.
- TABLE : SELECT, INSERT, UPDATE, DELETE, ALTER, INDEX, REFERENCES;( 외래 키 제약 조건을 걸어도 되)
- VIEW : SELECT, INSERT, UPDATE, DELETE;
- SEQUENCE : SELECT, ALTER
- PROCEDURE, FUNCTION PACKAGE : EXECUTE - 다른 소유자가 가지고 있는 거는 EXECUTE 권한이 필요하다
```bash
[oracle@oracle19c ~]$ sqlplus insa/oracle

SQL*Plus: Release 19.0.0.0.0 - Production on Fri Oct 16 03:56:23 2026
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

ERROR:
ORA-01045: user INSA lacks CREATE SESSION privilege; logon denied


<<SYS SESSION>>
GRANT CREATE SESSION TO insa;

SELECT * FROM dba_sys_privs
WHERE grantee = 'INSA';


<<insa SESSION>>
[oracle@oracle19c ~]$ sqlplus insa/oracle

SQL*Plus: Release 19.0.0.0.0 - Production on Fri Oct 16 03:59:38 2026
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.

ERROR:
ORA-28001: the password has expired


Changing password for insa
New password: <<- 새로운 패스워드 입력
Retype new password: <<- 새로운 패스워드 입력
Password changed

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0
```

#insa 유저가 테이블 생성하려면 필요한 권한
1. create table 시스템 권한 
2. tablespace 사용할 수 있는 쿼터값

```bash
<<insa 세션>>
INSA@ora19c> select * from user_ts_quotas;

TABLESPACE_NAME                     BYTES  MAX_BYTES     BLOCKS MAX_BLOCKS DRO
------------------------------ ---------- ---------- ---------- ---------- ---
INSA_TBS                                0         -1          0         -1 NO

CREATE TABLE insa.emp(id number, name varchar2(30));


INSA@ora19c> CREATE TABLE insa.emp (
    id NUMBER,
    name VARCHAR2(30)
);  2    3    4
CREATE TABLE insa.emp (
*
ERROR at line 1:
ORA-01031: insufficient privileges

#sys 세션 
GRANT CREATE TABLE TO insa; 

SELECT * FROM dba_sys_privs WHERE grantee = 'INSA';


SYS@ora19c> SELECT * FROM dba_sys_privs WHERE grantee = 'INSA';

GRAN PRIVILEGE                                ADM COM INH
---- ---------------------------------------- --- --- ---
INSA CREATE TABLE                             NO  NO  NO
INSA CREATE SESSION                           NO  NO  NO

#insa 세션

INSA@ora19c> SELECT * FROM user_sys_privs;

USERNAME        PRIVILEGE               ADM  COM  INH
--------------- ----------------------- ---- ---- ----
INSA            CREATE TABLE            NO   NO   NO
INSA            CREATE SESSION          NO   NO   NO

2 rows selected.


INSA@ora19c> select * from user_ts_quotas;

TABLESPACE_NAME                     BYTES  MAX_BYTES     BLOCKS MAX_BLOCKS DRO
------------------------------ ---------- ---------- ---------- ---------- ---
INSA_TBS                                0         -1          0         -1 NO


INSA@ora19c> SELECT default_tablespace FROM user_users;

DEFAULT_TABLESPACE
------------------------------
INSA_TBS

INSA@ora19c> CREATE TABLE insa.emp (
                 id NUMBER,
                 name VARCHAR2(30)
             );

Table created.

INSA@ora19c> SELECT table_name, tablespace_name FROM user_tables;

TABLE_NAME                     TABLESPACE_NAME
------------------------------ ------------------------------
EMP                            INSA_TBS


INSA@ora19c> insert into insa.emp(id,name) values(1,'scott');
#소유자라서 그냥 넣을 수 있다
1 row created.

INSA@ora19c> commit;

Commit complete.

INSA@ora19c> select * from insa.emp;

        ID NAME
---------- ------------------------------
         1 scott


<<sys 세션>> 
SYS@ora19c> CREATE USER insa_buha
IDENTIFIED BY oracle
DEFAULT TABLESPACE insa_tbs
TEMPORARY TABLESPACE insa_temp
QUOTA 1M ON insa_tbs;
User created.

SYS@ora19c> GRANT CREATE SESSION TO insa_buha;

Grant succeeded.

SYS@ora19c> SELECT * from dba_sys_privs where grantee = 'INSA_BUHA';

GRANTEE   PRIVILEGE                                ADM COM INH
--------- ---------------------------------------- --- --- ---
INSA_BUHA CREATE SESSION                           NO  NO  NO

<<INSA_BUHA SESSION>>
INSA_BUHA@ora19c> SELECT * FROM user_sys_privs;

USERNAME        PRIVILEGE               ADM  COM  INH
--------------- ----------------------- ---- ---- ----
INSA_BUHA       CREATE SESSION          NO   NO   NO

1 row selected.

INSA_BUHA@ora19c> select * from insa.emp;
select * from insa.emp
                   *
ERROR at line 1:
ORA-00942: table or view does not exist

<<INSA SESSION>>
INSA@ora19c> GRANT SELECT ON insa.emp TO insa_buha;

Grant succeeded.

#내가 받은 권한도 확인할 수 있고 내가 준 권한도 확인할 수 있다.

INSA@ora19c> SELECT * FROM user_tab_privs;

GRANTEE    OWNER    TABLE_NAME    GRANTOR    PRIVILEGE            GRA  HIE  COM  TYPE    INH
---------- -------- ------------- ---------- -------------------- ---- ---- ---- ------- ----
INSA_BUHA  INSA     EMP           INSA       SELECT               NO   NO   NO   TABLE   NO
PUBLIC     SYS      INSA          INSA       INHERIT PRIVILEGES   NO   NO   NO   USER    NO

2 rows selected.

#sys세션
SELECT *
FROM dba_tab_privs
WHERE grantee IN ('INSA', 'INSA_BUHA');
```
![](/assets/img/oracle-2026-10-16/11.png)
```bash
#인사 부하 세션
INSA_BUHA@ora19c> SELECT * FROM user_tab_privs;

GRANTEE    OWNER       TABLE_NAME    GRANTOR     PRIVILEGE           GRA HIE COM TYPE  INH
---------- ----------- ------------- ----------- ------------------- --- --- --- ----- ---
INSA_BUHA  INSA        EMP           INSA        SELECT              NO  NO  NO  TABLE NO
PUBLIC     SYS         INSA_BUHA     INSA_BUHA   INHERIT PRIVILEGES  NO  NO  NO  USER  NO


INSA_BUHA@ora19c> SELECT * FROM insa.emp;

        ID NAME
---------- ------------------------------
         1 scott
         

INSA_BUHA@ora19c> set serveroutput on
INSA_BUHA@ora19c> DECLARE
        v_rec insa.emp%rowtype;
BEGIN
        SELECT *
        INTO v_rec
        FROM insa.emp
        WHERE id = 1;

        DBMS_OUTPUT.PUT_LINE(v_rec.id||' '||v_rec.name);
END;
/ 
1 scott
```
- 익명블록 구조는 따로 권한이 필요없고 익명블록이 작동되는 테이블에 대한 권한만 있으면 된다.
```bash
INSA_BUHA@ora19c> insert into insa.emp(id, name) values(2,'liam');
insert into insa.emp(id, name) values(2,'liam')
                 *
ERROR at line 1:
ORA-01031: insufficient privileges

INSA_BUHA@ora19c> update insa.emp set name = 'liam' where id = 1;
update insa.emp set name = 'liam' where id = 1
            *
ERROR at line 1:
ORA-01031: insufficient privileges


INSA_BUHA@ora19c> delete from insa.emp where id = 1;
delete from insa.emp where id = 1
                 *
ERROR at line 1:
ORA-01031: insufficient privileges
```
-근데 insert권한이 없을때 하면 권한 없다고 뜨고 select권한 없을때 하면 테이블이 없다고 뜨자나 select권한 자체도 없는데 insert하면 어떻게 떠나? —\> 존재하지 않는 테이블이라고 뜬다.
```bash
#인사 세션
INSA@ora19c> GRANT INSERT,UPDATE, DELETE ON insa.emp TO insa_buha;

Grant succeeded.

#인사 부하세션

INSA_BUHA@ora19c> SELECT * FROM user_tab_privs;

GRANTEE    OWNER       TABLE_NAME    GRANTOR     PRIVILEGE           GRA HIE COM TYPE  INH
---------- ----------- ------------- ----------- ------------------- --- --- --- ----- ---
INSA_BUHA  INSA        EMP           INSA        DELETE              NO  NO  NO  TABLE NO
INSA_BUHA  INSA        EMP           INSA        INSERT              NO  NO  NO  TABLE NO
INSA_BUHA  INSA        EMP           INSA        SELECT              NO  NO  NO  TABLE NO
INSA_BUHA  INSA        EMP           INSA        UPDATE              NO  NO  NO  TABLE NO
PUBLIC     SYS         INSA_BUHA     INSA_BUHA   INHERIT PRIVILEGES  NO  NO  NO  USER  NO

INSA_BUHA@ora19c> insert into insa.emp(id, name) values(2,'liam');

1 row created.

INSA_BUHA@ora19c> update insa.emp set name = 'Liam' where id = 2;

1 row updated.

INSA_BUHA@ora19c> commit;

Commit complete.

INSA_BUHA@ora19c> select * from insa.emp;

        ID NAME
---------- ------------------------------
         1 scott
         2 Liam

INSA_BUHA@ora19c> delete from insa.emp where id = 2;

1 row deleted.

INSA_BUHA@ora19c> select * from insa.emp;

        ID NAME
---------- ------------------------------
         1 scott
         

<<INSA SESSION>>
create sequence insa.id_seq;

INSA@ora19c> create sequence insa.id_seq;
create sequence insa.id_seq
*
ERROR at line 1:
ORA-01031: insufficient privileges <<- create sequence 시스템권한이 없기 때문에 오류발생

<<SYS SESSION>>

SYS@ora19c> GRANT CREATE SEQUENCE TO insa;

Grant succeeded.

SYS@ora19c> select * from dba_sys_privs where grantee = 'INSA';

GRAN PRIVILEGE                                ADM COM INH
---- ---------------------------------------- --- --- ---
INSA CREATE TABLE                             NO  NO  NO
INSA CREATE SESSION                           NO  NO  NO
INSA CREATE SEQUENCE                          NO  NO  NO


<<insa session>>

INSA@ora19c> SELECT * FROM user_sys_privs;

USERNAME        PRIVILEGE               ADM  COM  INH
--------------- ----------------------- ---- ---- ----
INSA            CREATE SEQUENCE         NO   NO   NO
INSA            CREATE TABLE            NO   NO   NO
INSA            CREATE SESSION          NO   NO   NO


INSA@ora19c> CREATE SEQUENCE insa.id_seq;

Sequence created.


INSA@ora19c> SELECT * FROM user_sequences
             WHERE sequence_name = 'ID_SEQ';

SEQUENCE_NAME   MIN_VALUE    MAX_VALUE  INCREMENT_BY  C  O  CACHE_SIZE  LAST_NUMBER  S  E  S  S  K
--------------- ---------- ------------ ------------ -- -- ----------- ------------ -- -- -- -- --
ID_SEQ                  1   1.0000E+28            1  N  N          20            1  N  N  N  N  N


#insa 부하 세션
INSA_BUHA@ora19c> delete from insa.emp;

1 row deleted.

INSA_BUHA@ora19c> commit;

Commit complete.

INSA_BUHA@ora19c> select * from insa.emp;

no rows selected

INSA_BUHA@ora19c> INSERT INTO insa.emp(id, name)
                 VALUES(insa.id_seq.NEXTVAL, 'scott');  2
                 VALUES(insa.id_seq.NEXTVAL, 'scott')
                             *
ERROR at line 2:
ORA-00942: table or view does not exist <<- insa.id_seq 객체권한이 없기 때문에 오류발생

#insa 세션

INSA@ora19c> grant select on insa.id_seq TO insa_buha;

Grant succeeded.

INSA@ora19c> SELECT * FROM user_tab_privs;

GRANTEE    OWNER    TABLE_NAME    GRANTOR    PRIVILEGE           GRA HIE COM TYPE     INH
---------- -------- ------------- ---------- ------------------- --- --- --- -------- ---
INSA_BUHA  INSA     ID_SEQ        INSA       SELECT              NO  NO  NO  SEQUENCE NO
INSA_BUHA  INSA     EMP           INSA       SELECT              NO  NO  NO  TABLE    NO
INSA_BUHA  INSA     EMP           INSA       DELETE              NO  NO  NO  TABLE    NO
INSA_BUHA  INSA     EMP           INSA       INSERT              NO  NO  NO  TABLE    NO
INSA_BUHA  INSA     EMP           INSA       UPDATE              NO  NO  NO  TABLE    NO
PUBLIC     SYS      INSA          INSA       INHERIT PRIVILEGES  NO  NO  NO  USER     NO

6 rows selected.

#insa 부하 세션
INSA_BUHA@ora19c> SELECT *
                 FROM user_tab_privs
                 WHERE table_name = 'ID_SEQ';

GRANTEE    OWNER  TABLE_NAME  GRANTOR  PRIVILEGE  GRA  HIE  COM  TYPE      INH
---------  -----  ----------  -------  ---------  ---  ---  ---  --------  ---
INSA_BUHA  INSA   ID_SEQ      INSA     SELECT     NO   NO   NO   SEQUENCE  NO



INSA_BUHA@ora19c> select * from user_sequences;

no rows selected
                 
INSA_BUHA@ora19c> SELECT * FROM all_sequences
                 WHERE sequence_name = 'ID_SEQ';

SEQUENCE_OWNER  SEQUENCE_NAME  MIN_VALUE    MAX_VALUE  INCREMENT_BY C O CACHE_SIZE LAST_NUMBER S E S S K
--------------- -------------- ---------- ------------ ------------ - - ---------- ----------- - - - - -
INSA            ID_SEQ                  1   1.0000E+28            1 N N         20          21 N N N N N

1 row selected.                 

INSA_BUHA@ora19c> INSERT INTO insa.emp(id, name) VALUES(insa.id_seq.NEXTVAL, 'scott');

1 row created.

INSA_BUHA@ora19c> commit;

Commit complete.

INSA_BUHA@ora19c> select * from insa.emp;

        ID NAME
---------- ------------------------------
         1 scott

INSA_BUHA@ora19c> select insa.id_seq.currval from dual;

   CURRVAL
----------
         1



```
---
### 1. 초기 파라미터 파일과 Control File
- 데이터베이스 기동 시 초기 파라미터 파일(PFILE 또는 SPFILE)의 `CONTROL_FILES` 설정으로 Control File의 위치를 확인한다.
- **NOMOUNT:** 초기 파라미터 파일을 읽어 Instance를 기동한다. **MOUNT:** 지정된 경로의 Control File을 열어 데이터베이스를 연결한다.
- Control File 경로 확인과 변경 절차는 위의 Control File 이관 실습에 정리되어 있다.

### 2. SQL 실행 계획
- SQL의 Parsing 및 최적화 과정에서 실행 계획을 결정하고 **Execute 단계에서 그 계획을 사용해 SQL을 실행**한다.

### 3. Redo Entry와 Undo
- **Redo Entry:** 데이터 블록 등의 변경 사항을 복구할 수 있도록 기록하는 변경 정보다. Redo는 장애 발생 후 변경 내용을 재적용할 때 사용한다.
- **Undo:** 데이터 변경 전 상태를 재구성하기 위한 정보로 Rollback과 읽기 일관성(Read Consistency)에 사용한다. Undo 블록의 변경 내용도 Redo에 기록될 수 있다.

sqlplus \>`COL username FORMAT A15`
— 덜깨
## 오류 사례
### 4. HR 계정에서 동적 성능 뷰 조회 오류 (ORA-00942)
```sql
HR@ora19c> SELECT s.username, t.xidusn, t.ubafil, t.ubablk, t.used_ublk
             FROM v$session s, v$transaction t
             WHERE s.saddr = t.ses_addr;

ERROR at line 2:
ORA-00942: table or view does not exist
```
- **원인:** HR 사용자에게 `V$SESSION` 및 `V$TRANSACTION` 조회에 필요한 권한이 없어 발생한 오류다. 같은 조회를 SYS 계정으로 실행했을 때는 결과가 출력됐다.
- 사용자에 따라 데이터 딕셔너리와 동적 성능 뷰에 접근할 수 있는 권한이 다르다.
### 5. Tablespace 생성 시 파일 크기 오류 (ORA-03214)
```sql
SYS@ora19c> CREATE TABLESPACE insa_tbs
             DATAFILE '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf'
             SIZE 10 AUTOEXTEND ON;

ERROR at line 1:
ORA-03214: File Size specified is smaller than minimum required

SYS@ora19c> CREATE TABLESPACE insa_tbs
             DATAFILE '/u01/app/oracle/oradata/ORA19C/insa_tbs01.dbf'
             SIZE 10M AUTOEXTEND ON;

Tablespace created.
```
- **원인:** `SIZE 10`은 단위를 붙이지 않으면 10바이트이므로 데이터 파일의 최소 크기보다 작다.
- **해결:** `SIZE 10M`으로 수정하자 Tablespace가 생성됐다.
### 6. 급여 UPDATE 시 숫자 정밀도 초과 (ORA-01438)
```sql
HR@ora19c> UPDATE hr.employees
             SET salary = salary * 5000
             WHERE employee_id = 150;

ERROR at line 1:
ORA-01438: value larger than specified precision allowed for this column

HR@ora19c> UPDATE hr.employees
             SET salary = salary * 1.5
             WHERE employee_id = 150;

1 row updated.
```
- **원인:** 5000배 계산 결과가 `SALARY` 컬럼에서 허용하는 숫자 정밀도를 초과했다.
- **해결:** 1.5배로 변경한 UPDATE는 정상 실행됐다.
### 7. 시퀀스 INSERT 값 개수 오류 (ORA-00913)
```sql
INSA_BUHA@ora19c> INSERT INTO insa.emp(id,name)
                  VALUES(insa, id_seq.nextval, 'scott');

ERROR at line 1:
ORA-00913: too many values
```
- **원인:** INSERT 대상 컬럼은 `ID`와 `NAME`의 2개인데 `VALUES`에 3개 값을 지정했다.
- **올바른 형식:** 시퀀스 이름은 `insa.id_seq.NEXTVAL`처럼 소유자와 시퀀스 이름을 점으로 연결한다.
```sql
INSERT INTO insa.emp(id, name)
VALUES(insa.id_seq.NEXTVAL, 'scott');
```
- **주의:** 위 형식이 올바르더라도 `INSA_BUHA`가 `INSA.ID_SEQ`에 대한 `SELECT` 권한을 받기 전에는 별도로 `ORA-00942`가 발생했다. 권한 부여와 최종 성공 결과는 위쪽 시퀀스 실습에 이미 기록돼 있다.
