---
title: "Oracle 19c 테이블스페이스와 데이터파일 관리 실습 | 29일차"
date: 2026-10-15 00:00:00 +0900
categories: [Oracle]
tags: [Oracle 19c, 테이블스페이스, 데이터파일]
---

---
■  오라클은 데이터를 논리적으로 tablespace에 저장하고 물리적으로 datafile 저장된다.

■ tablespace
- tablespace는 하나 이상의 data file로 구성한다.
- 테이블스페이스는 세그먼트를 저장하는 논리적인 저장소이다.
```javascript
select * from dba_tablespaces;
```
![](/assets/img/oracle-2026-10-15/01.png)
CONTENTS :
	- PERMANENT: 영구적 부여된다
	- UNDO : 롤백이랑 읽기 일관성을 위해서 있는 것이다
	- TEMPORARY: 서버 프로세스가 생성하고 SMON이 해결한다.
MIN_EXTENTS:
	- 테이블을 테이블 스페이스에 생성하게 되면 무조건 extent를 무조건 하나 만든다
	- 그 크기는 INTIAL_EXTENT  크기로 만들어진다


```javascript
select * from dba_data_files;
select * from v$datafile;
```
![](/assets/img/oracle-2026-10-15/02.png)
- 데이터 파일은 컨트롤 파일이랑 딕셔너리 둘다 가지고 있다. 
- 데이터 파일을 컨트롤 파일이 가지고 있어야 오픈시점에 인스턴스를 올릴 수 있다. 
- **dba_ , user_ : 라고 적혀있는 애들은 오픈한 이후에 볼 수 있다**
- **v\$ :  노마운트 단계에서 볼 수 있다. 퍼포먼스 뷰라고 부른다.**

#system tablespace 
- 데이터베이스와 함께 생성한다.
- data dictionary table들이 생성되는 곳
- system undo segment를 포함하고 있다. 

#sysaux tablespace (10g) 
- system tablespace의 보조역할의 테이블스페이스이다.
- 성능진단(AWR)과 관련된 테이블이 저장되어 있는 곳

#non system tablespace
- 데이터베이스 관리가 더욱 유연해진다.
- undo, temporary, 일반 데이터를 저장하는 곳 (비즈니스 관련 데이터, index)
- 테이블스페이스는 업무별로 분리해서 사용한다. 

■ data file
- tablespace는 data file이라는 하나 이상의 파일로 구성되어 있다. 
- data file은 OS에 따르는 물리적 구조이다. 

■segment
- 데이터베이스에서 공간을 차지하는 객체를 말하며 데이터파일에 있는 공간을 사용한다. 
- 세그먼트는 테이블스페이스 안의 특정 논리적 저장 영역 구조에 할당된 영역이다. 
- 테이블, 인덱스, 언두, lob등 오라클이 제공하는 논리적 공간을 의미한다.
- 세그먼트는 테이블스페이스에 속한 여러 데이터파일로 확장할 수 있다. 
	- 하나의 세그먼트 파일이 데이터 공간이 부족할 경우를 대비해서 하나의 세그먼트 파일에 여러개의 extent를 붙여서 공간 부족을 방지한다.
![](/assets/img/oracle-2026-10-15/03.png)

■ extent
- 하나 이상의 extent로 segemet를 구성한다.
- extent는 연속된 oracle block으로 구성되어 있다.
- 세그먼트가 사용하는 공간
- 세그먼트에서 extent가 할당될때
	- create(생성)
	- extend(확장, 데이터가 입력)
	- alter(변경)
- 세그먼트에서 extent가 해제
	- drop(삭제)
	- alter(변경)
	- truncate(잘릴경우)

■ block
- 실제 데이터가 저장되는 곳
- 오라클 최소 I/O 단위
- 오라클이 사용할 수 있는 블록 크기는 2k, 4k, 8k, 16k, 32k
- 오라클 데이터베이에서 사용하는 기본 블록 크기는 db_block_size파라미터로 결정된다. (기본크기 8k)
- 오라클 블록은 하나 이상의 OS블록을 가지고 생성한다.

```javascript
SYS@ora19c> show parameter db_block_size

NAME                                 TYPE        VALUE
------------------------------------ ----------- ------------------------------
db_block_size                        integer     8192


#os 블락 크기 보는 법
SYS@ora19c> select max(lebsz) from x$kccle;

MAX(LEBSZ)
----------
       512

```

```sql
select * from v$log;

SELECT a.group#,
       b.sequence#,
       a.member,
       b.members,
       b.bytes / 1024 / 1024 AS mb,
       b.archived,
       b.status,
       b.first_change#,
       b.next_change#
FROM v$logfile a, v$log b
WHERE a.group# = b.group#
ORDER BY 1;
```
![](/assets/img/oracle-2026-10-15/04.png)

■ tablespace 생성
```sql
CREATE TABLESPACE userdata
DATAFILE '/u01/app/oracle/oradata/ORA19C/usersdata01.dbf' SIZE 5m;
```

![](/assets/img/oracle-2026-10-15/05.png)
- select \* from dba_tablespaces; 로 본 결과
- ALLOCATION_TYPE 이 SYSTEM이면 가변이고 공간은 계속 늘어나는데 줄어들지는 않는다.

![](/assets/img/oracle-2026-10-15/06.png)
- select \* from dba_data_files; 로 본 결과

```sql
create table hr.emp
tablespace userdata
as select * from hr.employees;

select * from dba_segments where segment_name = 'EMP';
```
![](/assets/img/oracle-2026-10-15/07.png)

```sql
select * from dba_extents where segment_name = 'EMP';
```
![](/assets/img/oracle-2026-10-15/08.png)
extent_id 번호는 무조건  0번 부터 시작한다.  block번호는 자기가 속해 있는 
Oracle B-tree 인덱스의 루트 블락=은 헤더블락(block_id) 다음에 있다.

#데이터파일의 크기를 수동으로 설정;
```sql
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf' 
RESIZE 10M;

select * from dba_data_files;

! ls -lh /u01/app/oracle/oradata/ORA19C/userdata01.dbf

-rw-r-----. 1 oracle oinstall 11M Oct 15 11:07 /u01/app/oracle/oradata/ORA19C/userdata01.dbf
```

```sql
-- 자동확장 기능 활성화 (autoextend on)
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/usersdata01.dbf' 
AUTOEXTEND ON;

SELECT * FROM dba_data_files;

-- 자동확장 기능 비활성화 (autoextend off)
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/usersdata01.dbf'
AUTOEXTEND OFF;
 
SELECT * FROM dba_data_files;
```
![YES면 활성화 된 상태이다](/assets/img/oracle-2026-10-15/09.png)

#테이블 스페이스 데이터파일 추가
```sql
alter tablespace userdata add datafile '/u01/app/oracle/oradata/ORA19C/usersdata02.dbf' 
size 5m;
SELECT * FROM dba_data_files;
```
![](/assets/img/oracle-2026-10-15/10.png)
- userdata02라는 파일을 userdata에 추가했다.
```sql
-- 자동확장 기능 활성화 (autoextend on maxsize 값)
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/usersdata02.dbf' 
AUTOEXTEND ON next 1m maxsize 10m;
ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/usersdata02.dbf' 
AUTOEXTEND ON next 2m maxsize unlimited;
SELECT * FROM dba_data_files;

SELECT * FROM dba_data_files;
-- 이거 unlitm면  oracle의 data file 크기 제한이 적용됨
```
- next 1m : 다음 확장시 1m 씩 늘어난다
![](/assets/img/oracle-2026-10-15/11.png)
- 1m씩 확장할때 마다 128 block씩 늘어난다

#테이블스페이스 삭제
```sql
drop tablespace userdata;

#테이블 스페이스에 파일이 있으면 오류 발생
명령의 46 행에서 시작하는 중 오류 발생 -
drop tablespace userdata
오류 보고 -
ORA-01549: 테이블스페이스가 비어있지 않으므로 INCLUDING CONTENTS 옵션을 사용해 주십시오
01549. 00000 -  "tablespace not empty, use INCLUDING CONTENTS option"
*Cause:    Tried to drop a non-empty tablespace
*Action:   To drop all the objects in the tablespace, use the INCLUDING
           CONTENTS option
           
           
 drop tablespace userdata including contents and datafiles;
 # including contents and datafiles: 논리적으로 속한 파일을 없애준다
 #물리적인 것도 없애준다. 
```

#extent  크기 결정
```sql
CREATE TABLESPACE userdata
DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf' 
SIZE 5m AUTOEXTEND ON
EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1m; 
```
![](/assets/img/oracle-2026-10-15/12.png)

#extent 크기 설정
- autoallocate : extent 크기는 처음 64k로 생성한 후 데이터가 입력되는 추세에 따라 오라클이 자동으로 크기를 결정한다.
- uniform : extent 크기는 1m 설정된다. 
- uniform size : extent 크기를 수동으로 설정, kb, mb 단위로 설정
- **AUTOALLOCATE는 Extent 관리 방식이고 AUTOEXTEND는 Data File 크기 관리 기능이다.**

```sql
SELECT * FROM dba_data_files;

select * FROM dba_tablespaces;

create table hr.emp
tablespace userdata
as select * from hr.employees;
```
![](/assets/img/oracle-2026-10-15/13.png)
기본은 1로 extent가 설정되어 있고
```sql
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;

```
![](/assets/img/oracle-2026-10-15/14.png)

넣으면 계속 늘어난다.
```sql
ROLLBACK; 
# ROLLBACK해도 block이 줄어들지 않는다

DELETE FROM hr.emp;
# 테이블을 delete해도 block크기는 줄어들지 않는다.

truncate table hr.emp;
drop table hr.emp;
#드랍 또는 truncate해야 extent크기가 줄어든다
```

```sql
drop tablespace userdata including contents and datafiles;
SELECT * FROM dba_data_files;

CREATE TABLESPACE userdata
DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf'
SIZE 5M AUTOEXTEND ON
EXTENT MANAGEMENT LOCAL AUTOALLOCATE;

SELECT * FROM dba_data_files;
select * FROM dba_tablespaces;

create table hr.emp
tablespace userdata
as select * from hr.employees;

select * from dba_segments where segment_name = 'EMP';
select * from dba_extents where segment_name = 'EMP';

insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;

select * from dba_extents where segment_name = 'EMP';

ROLLBACK;
```
![](/assets/img/oracle-2026-10-15/15.png)
- AUTOALLOCATE라고 설정해서 처음에는64k로 만들어진다
![](/assets/img/oracle-2026-10-15/16.png)
- 데이터를 계속 넣다 보면 오라클이 알아서 extents 용량을 조절한다.
```sql
truncate table hr.emp;
select * from dba_extents where segment_name = 'EMP';
```
![](/assets/img/oracle-2026-10-15/17.png)
- 알아서 조절하는 모습을 볼 수 있다.

#자동 확장 끄면
```sql
drop tablespace userdata including contents and datafiles;
SELECT * FROM dba_data_files;

CREATE TABLESPACE userdata
DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf' SIZE 5M
EXTENT MANAGEMENT LOCAL AUTOALLOCATE;

SELECT * FROM dba_data_files;
select * FROM dba_tablespaces;

create table hr.emp
tablespace userdata
as select * from hr.employees;

select * from dba_segments where segment_name = 'EMP';
select * from dba_extents where segment_name = 'EMP';

insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;
insert into hr.emp select * from hr.emp;

select * from dba_segments where segment_name = 'EMP';
select * from dba_extents where segment_name = 'EMP';

명령의 147 행에서 시작하는 중 오류 발생 -
insert into hr.emp select * from hr.emp
오류 보고 -
ORA-01653: HR.EMP 테이블을 128(으)로 USERDATA 테이블스페이스에서 확장할 수 없습니다
```

#다른 창에 확정 기능을 켜야한다. 
```sql
ALTER DATABASE DATAFILE
'/u01/app/oracle/oradata/ORA19C/userdata01.dbf'
AUTOEXTEND ON NEXT 1M MAXSIZE 100M;

ALTER DATABASE DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf' 
AUTOEXTEND ON next 2m maxsize unlimited;
```

- Private strand flush not complete
```bash
select d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled, d.online_status
from v$datafile v, dba_data_files d
where v.file# = d.file_id;
```

- **Private strand flush not complete**: Redo Log Switch 시 Private Redo Strand에 남아 있는 Redo 정보의 기록이 완료되지 않아 Log Switch가 잠시 대기하는 현상이다. Redo 기록이 완료되면 다음 Redo Log Group으로 전환된다. 일반적으로 정상적인 대기 현상이지만 반복적으로 오래 지연되면 성능 점검이 필요하다.

■ 테이블스페이스 read only
- read only 테이블스페이스안에 있는 테이블에 대해서는 읽기만 가능하다.
- DML 작업은 불허
- DML 작업은 불허,  CREATE TABLE,  ALTER(DROP COLUMN), TRUNCATE TABLE 불허
- partial checkpoint 발생

```sql
select d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled, d.online_status
from v$datafile v, dba_data_files d
where v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3795568 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3795568 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3795568 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3795568 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3795568 READ WRITE ONLINE
      
SYS@ora19c> alter tablespace userdata read only;


Tablespace altered.

select d.file_id, d.tablespace_name, d.file_name, v.checkpoint_change#, v.enabled, d.online_status
from v$datafile v, dba_data_files d
where v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3795568 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3795568 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3795568 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3795568 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3795732 READ ONLY  ONLINE


select * from hr.emp; -- 얘는 됨

SYS@ora19c> update hr.emp set salary = salary * 1.1 where employee_id = 200;

update hr.emp set salary = salary * 1.1 where employee_id = 200
       *
ERROR at line 1:
ORA-00372: file 5 cannot be modified at this time
ORA-01110: data file 5: '/u01/app/oracle/oradata/ORA19C/userdata01.dbf'


delete from hr.emp where department_id = 20;

오류 보고 -
ORA-00372: 파일 5는 지금 수정될 수 없습니다
ORA-01110: 5 데이터 파일: '/u01/app/oracle/oradata/ORA19C/userdata01.dbf'


insert into hr.emp select * from hr.employees;

오류 보고 -
ORA-00372: 파일 5는 지금 수정될 수 없습니다
ORA-01110: 5 데이터 파일: '/u01/app/oracle/oradata/ORA19C/userdata01.dbf'


create table hr.dept
tablespace userdata
as select * from hr.departments;

오류 보고 -
ORA-01647: 'USERDATA' 테이블스페이스는 읽기 전용입니다. 영역을 할당할 수 없습니다.
01647. 00000 - "tablespace '%s' is read-only, cannot allocate space in it"
*Cause: Tried to allocate space in a read-only tablespace
*Action: Create the object in another tablespace 


SYS@ora19c> alter table hr.emp modify last_name varchar2(50);

Table altered.
#딕셔너리에서 col만 수정하기 때문에 가능하다.

SYS@ora19c> alter table hr.emp drop column email;
alter table hr.emp drop column email
*
ERROR at line 1:
ORA-12985: tablespace 'USERDATA' is read only, cannot drop column


alter table hr.emp add dept_name varchar2(30); #column drop은 안되고 추가는 된다.

Table HR.EMP이(가) 변경되었습니다.
#alter할때는 딕셔너리에만 넣기 때문에 가능하다.

SYS@ora19c> alter table hr.emp drop column dept_name;
alter table hr.emp drop column dept_name
*
ERROR at line 1:
ORA-12985: tablespace 'USERDATA' is read only, cannot drop column
#col을 drop하는 건 또 안된다


SYS@ora19c> truncate table hr.emp;
truncate table hr.emp
                  *
ERROR at line 1:
ORA-00372: file 5 cannot be modified at this time
ORA-01110: data file 5: '/u01/app/oracle/oradata/ORA19C/userdata01.dbf'

SYS@ora19c> drop table hr.emp purge;

Table dropped.

#truncate는 안되고 drop은 또 되네
```
#truncate랑 drop은 partial checkpoint 발생한다.
- redo only모드로 바꾸는 시점에 더티 버퍼가 전부 내려가서 체크포인트는 발생할 필요가 없다 . 바꾸는 시점에서 checkpoint가 발생해서

#테이블 스페이스 READ WRITE 
```sql
ALTER TABLESPACE 테이블스페이스이름 READ WRITE;

alter tablespace userdata read write

-- 1. READ ONLY 변경 전 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3795568 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3795568 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3795568 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3795568 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3795568 READ WRITE ONLINE


-- 2. USERDATA를 READ ONLY로 변경

SYS@ora19c> ALTER TABLESPACE userdata READ ONLY;

Tablespace altered.


-- 3. READ ONLY 변경 후 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3795568 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3795568 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3795568 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3795568 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3795732 READ ONLY  ONLINE




-- 4. USERDATA를 READ WRITE로 변경

SYS@ora19c> ALTER TABLESPACE userdata READ WRITE;

Tablespace altered.


-- 5. READ WRITE 변경 후 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3796292 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3797454 READ WRITE ONLINE

```
- READ WRITE로 바뀔때도 checkpoint가 발생한다.

■ 테이블스페이스 offline
- 테이블스페이스에 속한 세그먼트를 사용할 수 없도록한다.
- partial checkpoint
- offline으로 설정할 수 없는 테이블스페이스
	- system tablespace
	- 사용중인 undo tablespace
	- temp temporary tablespace

```sql
ALTER TABLESPACE 테이블스페이스 이름 OFFLINE [normal | temporary | immediate];
         normal : 기본값, partial checkpoint 발생
         temporary : 가능한 데이터파일에 속한 dirty buffer만 디스크로 쓰는 작업을 수행
         immediate : 체크포인트 발생하지 않습니다. 단 archivelog mode에서만 사용, 복구진행해야한다
         
SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3796292 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3797454 READ WRITE ONLINE



-- OFFLINE NORMAL 명시 형태: ALTER TABLESPACE userdata OFFLINE NORMAL;
-- NORMAL을 생략해도 기본값인 OFFLINE NORMAL로 동작한다.
SYS@ora19c> ALTER TABLESPACE userdata OFFLINE;

Tablespace altered.

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3796292 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3797841 DISABLED   OFFLINE
      
      
      
-- 1. OFFLINE 상태에서 CREATE TABLE 실행

SYS@ora19c> CREATE TABLE hr.dept
             TABLESPACE userdata
             AS SELECT * FROM hr.departments;

ERROR at line 3:
ORA-01542: tablespace 'USERDATA' is offline, cannot allocate space in it

원인:
USERDATA가 OFFLINE 상태이므로
새로운 테이블에 필요한 공간을 할당할 수 없다.


-- 2. SYSTEM Tablespace OFFLINE 시도

SYS@ora19c> ALTER TABLESPACE system OFFLINE;

ERROR at line 1:
ORA-01541: system tablespace cannot be brought offline; shut down if necessary

원인:
SYSTEM Tablespace는 데이터베이스 운영에 필수적이므로
OFFLINE 상태로 변경할 수 없다.


-- 3. UNDOTBS1 Tablespace OFFLINE 시도

SYS@ora19c> ALTER TABLESPACE undotbs1 OFFLINE;


ERROR at line 1:
ORA-30042: Cannot offline the undo tablespace

원인:
UNDO Tablespace는 트랜잭션과 복구에 사용되므로
일반적인 ALTER TABLESPACE OFFLINE 명령을 사용할 수 없다.


alter tablespace temp offline;

명령의 188 행에서 시작하는 중 오류 발생 -
alter tablespace temp offline
오류 보고 -
ORA-03217: TEMPORARY TABLESPACE의 변경하기 위한 옵션이 부적당합니다
03217. 00000 -  "invalid option for alter of TEMPORARY TABLESPACE"
*Cause:    invalid option for alter of temporary tablespace was specified
*Action:   Specify one of the valid options
           TEMPFILE ONLINE, TEMPFILE OFFLINE
           
           
 
SELECT * FROM dba_temp_files;         


-- 1. Data File 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3798099 DISABLED   OFFLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3797841 DISABLED   OFFLINE


-- 2. SYSAUX와 USERDATA를 ONLINE으로 변경

SYS@ora19c> ALTER TABLESPACE sysaux ONLINE;

Tablespace altered.

SYS@ora19c> ALTER TABLESPACE userdata ONLINE;

Tablespace altered.


-- 3. ONLINE 변경 후 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3798194 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3798228 READ WRITE ONLINE

SELECT * FROM hr.employees;
```
- 데이터 이관 작업 할때도 offline모드로 바꿔야한다.

```sql
-- 1. 기존 USERDATA Tablespace 삭제
DROP TABLESPACE userdata INCLUDING CONTENTS AND DATAFILES;


-- 2. USERDATA Tablespace 생성
CREATE TABLESPACE userdata
DATAFILE '/home/oracle/userdata01.dbf' SIZE 5M AUTOEXTEND ON
EXTENT MANAGEMENT LOCAL AUTOALLOCATE;


-- 3. Data File 및 Tablespace 정보 조회
SELECT * FROM dba_data_files;
SELECT * FROM dba_tablespaces;


-- 4. USERDATA Tablespace에 HR.EMP 테이블 생성
CREATE TABLE hr.emp
TABLESPACE userdata
AS SELECT * FROM hr.employees;


-- 5. EMP 테이블의 행 개수 조회
SELECT COUNT(*) FROM hr.emp;

SYS@ora19c> SELECT f.file_name
FROM dba_extents e, dba_data_files f
WHERE e.file_id = f.file_id
AND e.segment_name = 'EMP'
AND e.owner = 'HR';

FILE_NAME
--------------------------------------------------------------------------------
/home/oracle/userdata01.dbf


```

■ 데이터파일 이관 작업
```sql
/home/oracle/userdata01.dbf -> /u01/app/oracle/oradata/ORA19C/userdata01.dbf
```

**1. 테이블스페이스 OFFLINE NORMAL**
```sql
SYS@ora19c> alter tablespace userdata offline normal;

Tablespace altered.


SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3798194 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /home/oracle/userdata01.dbf                                 3800683 DISABLED   OFFLINE

```

--- 여기
**OFFLINE 상태에서 HR.EMP Data File 경로 조회**
```sql
SELECT f.file_name
FROM dba_extents e, dba_data_files f
WHERE e.file_id = f.file_id
AND e.segment_name = 'EMP'
AND e.owner = 'HR';
```
```plain text
no rows selected
```
- USERDATA를 OFFLINE NORMAL로 변경한 뒤 이 실습에서는 해당 조회가 0행을 반환했다. OFFLINE인 모든 테이블스페이스에서 항상 0행이 반환된다는 뜻은 아니다.

**2. 물리적 데이터파일 이동**<br>
```sql
SYS@ora19c> ! mv -v /home/oracle/userdata01.dbf /u01/app/oracle/oradata/ORA19C/userdata01.dbf

‘/home/oracle/userdata01.dbf’ -> ‘/u01/app/oracle/oradata/ORA19C/userdata01.dbf’
removed ‘/home/oracle/userdata01.dbf’

SYS@ora19c> ! ls /u01/app/oracle/oradata/ORA19C/userdata01.dbf
/u01/app/oracle/oradata/ORA19C/userdata01.dbf


```

**3. 기존파일을 새로운 파일 위치로 수정**
```sql
ALTER TABLESPACE 테이블스페이스이름 RENAME DATAFILE 'OLD' TO 'NET';

alter tablespace userdata rename datafile '/home/oracle/userdata01.dbf' to '/u01/app/oracle/oradata/ORA19C/userdata01.dbf';
-- 1. USERDATA Data File 경로 변경

SYS@ora19c> ALTER TABLESPACE userdata
             RENAME DATAFILE '/home/oracle/userdata01.dbf'
             TO '/u01/app/oracle/oradata/ORA19C/userdata01.dbf';

Tablespace altered.


-- 2. Data File 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3798194 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3800683 DISABLED   OFFLINE

```
**4. 테이블스페이스 ONLINE 확인**
```sql
-- 1. USERDATA Tablespace ONLINE 변경

SYS@ora19c> ALTER TABLESPACE userdata ONLINE;

Tablespace altered.


-- 2. ONLINE 변경 후 Data File 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3798194 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /u01/app/oracle/oradata/ORA19C/userdata01.dbf               3801814 READ WRITE ONLINE


-- 3. HR.EMP 테이블 데이터 확인

SYS@ora19c> SELECT COUNT(*) FROM hr.emp;

  COUNT(*)
----------
       107


```

- 데이터 이관을 할때 데이터베이스를 마운트까지만 올리고 나서 데이터를 이관한다. 

# 다시 원래대로 되돌리는 거
```sql
■ USERDATA Data File 경로 변경 및 ONLINE 복구 실습

-- 0. 먼저 해당 tablespace offline으로 만들기
SYS@ora19c> alter tablespace userdata offline normal;

Tablespace altered.


-- 1. Linux에서 Data File 물리적 위치 변경

SYS@ora19c> ! mv -v /u01/app/oracle/oradata/ORA19C/userdata01.dbf /home/oracle/userdata01.dbf

‘/u01/app/oracle/oradata/ORA19C/userdata01.dbf’ -> ‘/home/oracle/userdata01.dbf’
removed ‘/u01/app/oracle/oradata/ORA19C/userdata01.dbf’


-- 2. 이동한 Data File 확인

SYS@ora19c> ! ls /home/oracle/userdata01.dbf

/home/oracle/userdata01.dbf


-- 3. Oracle에 등록된 Data File 경로 변경

SYS@ora19c> ALTER TABLESPACE userdata
             RENAME DATAFILE '/u01/app/oracle/oradata/ORA19C/userdata01.dbf'
             TO '/home/oracle/userdata01.dbf';

Tablespace altered.


-- 4. Data File 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3798194 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /home/oracle/userdata01.dbf                                 3801999 DISABLED   OFFLINE


-- 5. USERDATA Tablespace ONLINE 변경

SYS@ora19c> ALTER TABLESPACE userdata ONLINE;

Tablespace altered.


-- 6. ONLINE 변경 후 상태 확인

SYS@ora19c> SELECT d.file_id, d.tablespace_name, d.file_name,
                   v.checkpoint_change#, v.enabled, d.online_status
             FROM v$datafile v, dba_data_files d
             WHERE v.file# = d.file_id;

FILE_ID TABLESPACE_NAME FILE_NAME                                        CHECKPOINT_CHANGE# ENABLED    ONLINE_STATUS
------- --------------- ------------------------------------------------ ------------------ ---------- -------------
      1 SYSTEM          /u01/app/oracle/oradata/ORA19C/system01.dbf                 3796292 READ WRITE SYSTEM
      3 SYSAUX          /u01/app/oracle/oradata/ORA19C/sysaux01.dbf                 3798194 READ WRITE ONLINE
      4 UNDOTBS1        /u01/app/oracle/oradata/ORA19C/undotbs01.dbf                3796292 READ WRITE ONLINE
      7 USERS           /u01/app/oracle/oradata/ORA19C/users01.dbf                  3796292 READ WRITE ONLINE
      5 USERDATA        /home/oracle/userdata01.dbf                                 3802106 READ WRITE ONLINE


-- 7. HR.EMP 테이블 데이터 확인

SYS@ora19c> SELECT COUNT(*) FROM hr.emp;

  COUNT(*)
----------
       107

```

**#모든 데이터파일 이관**
…. /\*.dbf → /home/oracle/userdata

```sql
SYS@ora19c> !
[oracle@oracle19c ~]$ pwd
/home/oracle
[oracle@oracle19c ~]$ mkdir userdata
[oracle@oracle19c ~]$ ls
control.sql  Downloads                     Music     STARTUP    userdata01.dbf
Desktop      initora19c.ora                Pictures  Templates  Videos
Documents    LINUX.X64_193000_db_home.zip  Public    userdata
[oracle@oracle19c ~]$ cd userdata/
[oracle@oracle19c userdata]$ pwd
/home/oracle/userdata
[oracle@oracle19c userdata]$

#mv -v를 간단가헤 만들기 위해서 사

SELECT 'mv -v ' || name || ' /home/oracle/userdata'
FROM v$datafile;

SELECT 'mv -v ' || name || ' /home/oracle/userdata'
FROM v$tempfile;

mv -v /u01/app/oracle/oradata/ORA19C/system01.dbf /home/oracle/userdata
mv -v /u01/app/oracle/oradata/ORA19C/sysaux01.dbf /home/oracle/userdata
mv -v /u01/app/oracle/oradata/ORA19C/undotbs01.dbf /home/oracle/userdata
mv -v /home/oracle/userdata01.dbf /home/oracle/userdata
mv -v /u01/app/oracle/oradata/ORA19C/users01.dbf /home/oracle/userdata

mv -v /u01/app/oracle/oradata/ORA19C/temp01.dbf /home/oracle/userdata

#데이터베이스 레벨에서 데이터파일 이관
alter database rename file '기존데이터파일' to '새로운데이터파';

alter database rename file '/u01/app/oracle/oradata/ORA19C/system01.dbf' to '/home/oracle/userdata/system01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/sysaux01.dbf' to '/home/oracle/userdata/sysaux01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf' to '/home/oracle/userdata/undotbs01.dbf';
alter database rename file '/home/oracle/userdata01.dbf' to '/home/oracle/userdata/userdata01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/users01.dbf' to '/home/oracle/userdata/users01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/temp01.dbf' to '/home/oracle/userdata/temp01.dbf';
```

**1. 데이터베이스 정상적인 종료**
```plain text
shutdown immediate
```

**2. 모든 데이터파일을 새로운 디스크 위치로 이동**
```bash
mv -v /u01/app/oracle/oradata/ORA19C/system01.dbf /home/oracle/userdata
mv -v /u01/app/oracle/oradata/ORA19C/sysaux01.dbf /home/oracle/userdata
mv -v /u01/app/oracle/oradata/ORA19C/undotbs01.dbf /home/oracle/userdata
mv -v /home/oracle/userdata01.dbf /home/oracle/userdata
mv -v /u01/app/oracle/oradata/ORA19C/users01.dbf /home/oracle/userdata
mv -v /u01/app/oracle/oradata/ORA19C/temp01.dbf /home/oracle/userdata

ls /home/oracle/userdata
```

**3. 데이터베이스를 MOUNT까지만 열기**
```bash
startup mount

select instance_name, status from v$instance;
```

**4. 기존 데이터파일 이름을 새로운 데이터 파일 이름으로 수정**
```bash
select name from v$datafile;
select name from v$tempfile;

alter database rename file '/u01/app/oracle/oradata/ORA19C/system01.dbf' to '/home/oracle/userdata/system01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/sysaux01.dbf' to '/home/oracle/userdata/sysaux01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf' to '/home/oracle/userdata/undotbs01.dbf';
alter database rename file '/home/oracle/userdata01.dbf' to '/home/oracle/userdata/userdata01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/users01.dbf' to '/home/oracle/userdata/users01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/temp01.dbf' to '/home/oracle/userdata/temp01.dbf';

select name from v$datafile;
select name from v$tempfile;
```

**5. 데이터베이스 OPEN**
```bash
alter database open
```

#모든 데이터파일 이관 실습
```bash

SYS@ora19c> shutdown immediate
Database closed.
Database dismounted.
ORACLE instance shut down.

SYS@ora19c> !
[oracle@oracle19c ~]$ mv -v /u01/app/oracle/oradata/ORA19C/system01.dbf /home/oracle/userdata
‘/u01/app/oracle/oradata/ORA19C/system01.dbf’ -> ‘/home/oracle/userdata/system01.dbf’
removed ‘/u01/app/oracle/oradata/ORA19C/system01.dbf’
[oracle@oracle19c ~]$ mv -v /u01/app/oracle/oradata/ORA19C/sysaux01.dbf /home/oracle/userdata
‘/u01/app/oracle/oradata/ORA19C/sysaux01.dbf’ -> ‘/home/oracle/userdata/sysaux01.dbf’
removed ‘/u01/app/oracle/oradata/ORA19C/sysaux01.dbf’
[oracle@oracle19c ~]$ mv -v /u01/app/oracle/oradata/ORA19C/undotbs01.dbf /home/oracle/userdata
‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’ -> ‘/home/oracle/userdata/undotbs01.dbf’
removed ‘/u01/app/oracle/oradata/ORA19C/undotbs01.dbf’
[oracle@oracle19c ~]$ mv -v /home/oracle/userdata01.dbf /home/oracle/userdata
‘/home/oracle/userdata01.dbf’ -> ‘/home/oracle/userdata/userdata01.dbf’
[oracle@oracle19c ~]$ mv -v /u01/app/oracle/oradata/ORA19C/users01.dbf /home/oracle/userdata
‘/u01/app/oracle/oradata/ORA19C/users01.dbf’ -> ‘/home/oracle/userdata/users01.dbf’
removed ‘/u01/app/oracle/oradata/ORA19C/users01.dbf’
[oracle@oracle19c ~]$ mv -v /u01/app/oracle/oradata/ORA19C/temp01.dbf /home/oracle/userdata
‘/u01/app/oracle/oradata/ORA19C/temp01.dbf’ -> ‘/home/oracle/userdata/temp01.dbf’
removed ‘/u01/app/oracle/oradata/ORA19C/temp01.dbf’

[oracle@oracle19c ~]$ ls /home/oracle/userdata
sysaux01.dbf  temp01.dbf     userdata01.dbf
system01.dbf  undotbs01.dbf  users01.dbf
[oracle@oracle19c ~]$ exit
exit

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

SYS@ora19c> select name from v$datafile;
NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/system01.dbf
/u01/app/oracle/oradata/ORA19C/sysaux01.dbf
/u01/app/oracle/oradata/ORA19C/undotbs01.dbf
/home/oracle/userdata01.dbf
/u01/app/oracle/oradata/ORA19C/users01.dbf

SYS@ora19c> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
/u01/app/oracle/oradata/ORA19C/temp01.dbf



SYS@ora19c> alter database rename file '/u01/app/oracle/oradata/ORA19C/temp01.dbf' to '/home/oracle/userdata/temp01.dbf';

Database altered.

SYS@ora19c> alter database rename file '/u01/app/oracle/oradata/ORA19C/system01.dbf' to '/home/oracle/userdata/system01.dbf';

Database altered.

SYS@ora19c> alter database rename file '/u01/app/oracle/oradata/ORA19C/sysaux01.dbf' to '/home/oracle/userdata/sysaux01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/undotbs01.dbf' to '/home/oracle/userdata/undotbs01.dbf';
alter database rename file '/home/oracle/userdata01.dbf' to '/home/oracle/userdata/userdata01.dbf';
alter database rename file '/u01/app/oracle/oradata/ORA19C/users01.dbf' to '/home/oracle/userdata/users01.dbf';
Database altered.

SYS@ora19c>
Database altered.

SYS@ora19c>
Database altered.

SYS@ora19c>
Database altered.

SYS@ora19c> select name from v$datafile;

NAME
--------------------------------------------------------------------------------
/home/oracle/userdata/system01.dbf
/home/oracle/userdata/sysaux01.dbf
/home/oracle/userdata/undotbs01.dbf
/home/oracle/userdata/userdata01.dbf
/home/oracle/userdata/users01.dbf

SYS@ora19c> select name from v$tempfile;

NAME
--------------------------------------------------------------------------------
/home/oracle/userdata/temp01.dbf




SYS@ora19c> alter database open;

Database altered.

```



---

## 10월 15일 수업 메모 정리

### 1. Oracle 데이터베이스 객체와 저장 공간
- 수업 메모에 나온 데이터베이스 객체: **테이블, 뷰, SYNONYM(동의어), 인덱스, 프로시저, 함수, 트리거, 패키지**.
- **SYNONYM(시노님, 동의어)**은 테이블이나 뷰 같은 기존 데이터베이스 객체에 다른 이름을 부여하는 Oracle 객체이며 자체적으로 데이터를 저장하는 Segment는 갖지 않는다.
- 익명 블록은 이름을 가진 저장 객체로 등록되는 프로시저나 함수와 구분된다.
- 데이터 저장 공간이 필요한 객체는 **Segment**를 사용한다. 테이블과 인덱스 등이 대표적이며 뷰와 동의어 등 모든 객체가 각각 데이터 Segment를 갖는 것은 아니다.
- 수업 중 그린 데이터베이스 구조 그림:
![](/assets/img/oracle-2026-10-15/18.png)

### 2. Segment와 Extent 및 Block의 관계
- Extent의 도입 이유는 **Block과 Segment의 저장 공간을 효율적으로 관리하기 위해서**라고 정리했다.
- 논리적 저장 구조를 **Tablespace → Segment → Extent → Oracle Block** 순서
- Extent는 연속된 Oracle Block을 묶은 공간으로 Segment에 할당된다. 데이터 증가에 따라 Segment에 Extent가 추가로 할당될 수 있다.

### 3. DML과 Undo 및 Redo
- DML로 데이터를 변경할 때 **Undo 정보와 Redo 정보**가 생성
- Undo의 주된 용도는 **Rollback과 읽기 일관성(Read Consistency)** 유지다.
- 변경 작업에서는 변경 전 상태를 복원할 수 있는 Undo 정보가 필요하며 Undo 블록의 변경 사항도 Redo에 기록될 수 있다.
- DML 트랜잭션이 시작될 때 Undo 영역을 확보한다

### 4. DDL과 Data Dictionary 및 SYSTEM Tablespace
- DDL을 실행하면 테이블 정의 등의 **Data Dictionary 정보가 변경**
- DDL 수행 과정에서 Dictionary에 대한 내부 DML이 발생할 수 있다
- SYSTEM Tablespace에는 시스템 트랜잭션을 처리하기 위한 전용 Rollback Segment가 존재한다. 데이터베이스 생성 시 자동으로 생성되며 일반 사용자 트랜잭션에서 사용하는 UNDO Tablespace와 구분된다.

### 5. SMON과 Instance Recovery
- SMON(System Monitor): 인스턴스가 비정상 종료된 후 재시작할 때 Instance Recovery를 수행하는 백그라운드 프로세스다. Oracle은 Redo를 이용해 변경 내용을 복구(Roll Forward)하고 Undo를 이용해 미완료 트랜잭션을 취소(Rollback)한다.

### 6. 데이터베이스의 논리적 구조와 물리적 구조
- **데이터베이스 구조:** Oracle은 논리적 구조와 물리적 구조로 구분된다. 논리적 구조는 Tablespace → Segment → Extent → Block 순서로 구성되며 데이터를 논리적으로 관리한다. 물리적 구조는 Data File과 Control File 및 Online Redo Log File 등 실제 디스크에 저장되는 파일로 구성된다. Tablespace는 하나 이상의 Data File과 연결된다.
