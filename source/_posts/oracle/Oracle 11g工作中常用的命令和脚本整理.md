---
title: Oracle Database 11g工作中常用的命令和脚本整理
categories:
- 常用命令与脚本
tags:
- oracle
- scripts
---

## 设置Oracle用户密码永不过期
``` perl
alter profile default limit PASSWORD_LIFE_TIME unlimited;

# 查看设置
select * from dba_profiles where resource_name like 'PASSWORD_LIFE_TIME%';
```

## 设置Oracle用户登陆失败次数限制
``` perl
# 设置登陆失败次数限制为100，错误登陆数超过100会导致用户被锁
alter profile default limit FAILED_LOGIN_ATTEMPTS 100;

# 无限制
alter profile default limit FAILED_LOGIN_ATTEMPTS unlimited;

# 查看登陆失败限制设置
select * from dba_profiles where resource_name like 'FAILED_LOGIN_ATTEMPTS%';
```

<!-- more -->

## 系统进程PID查找对应SQLTEXT
``` sql
SELECT sql_text FROM v$sqltext a 
  WHERE (a.hash_value, a.address) IN (SELECT DECODE (sql_hash_value,0, prev_hash_value,sql_hash_value),
        DECODE (sql_hash_value, 0, prev_sql_addr, sql_address)
    FROM v$session b WHERE b.paddr = (SELECT addr FROM v$process c WHERE c.spid =&pid)) ORDER BY piece ASC;
```

## 查找长事务对应的SQLTEXT
``` sql
with ltr as ( 
select to_char(sysdate,'YYYYMMDDHH24MISS') TM, 
       s.sid, 
       s.sql_id, 
       s.sql_child_number, 
       s.prev_sql_id, 
       xid, 
       to_char(t.start_date,'YYYYMMDDHH24MISS') start_time, 
       e.TYPE,e.block, 
       e.ctime, 
       decode(e.CTIME, 0, (sysdate - t.start_date) * 3600*24, e.ctime) el_second 
  from v$transaction t, v$session s,v$transaction_enqueue e
 where t.start_date <= sysdate - interval '200' second
   and t.addr = s.taddr 
   and t.addr = e.addr(+) ) 
  select ltr.* , (select q1.sql_text from v$sql q1 where ltr.prev_sql_id = q1.sql_id(+)
   and rownum = 1) prev_sql_text , 
  (select q1.sql_text from v$sql q1 where ltr.sql_id = q1.sql_id(+) 
   and ltr.sql_child_number = q1.CHILD_NUMBER(+)) sql_text 
   from ltr ltr;
```

## 永久设置sql*plus的环境变量
``` perl
echo "set pagesize 9999" >> $ORACLE_HOME/sqlplus/admin/glogin.sql
echo "set line 150" >> $ORACLE_HOME/sqlplus/admin/glogin.sql
echo "set long 5000" >> $ORACLE_HOME/sqlplus/admin/glogin.sql
```

## 查找非系统用户中无主键表
``` sql
select distinct at.TABLE_NAME, at.OWNER, at.NUM_ROWS
from 
  (SELECT owner,table_name FROM all_tables WHERE owner in 
   (select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE','DBSNMP','SYSMAN'))
MINUS
  SELECT owner,table_name FROM all_constraints WHERE owner in 
   (select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE','DBSNMP','SYSMAN')) 
  AND constraint_type = 'P' ) vn, all_tables at
where vn.TABLE_NAME=at.TABLE_NAME and vn.OWNER=at.OWNER and at.TABLE_NAME not like '%$%' order by 2,1;
```

## 查询有Object的schema
``` sql
select OWNER,count(OBJECT_NAME) OBJECT_NUM from dba_objects where OWNER in
    (select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE')) and OBJECT_TYPE='TABLE'
    group by OWNER;
```

## 动态性能视图授权普通用户查看权限
``` sql
grant select on v_$session to <schema>;
grant select on v_$locked_object to <schema>;
```

## 查看并kill掉锁对象
``` sql
select * from v$locked_object;
select object_name,object_type from dba_objects where object_id=&object_id;
select sid, serial#, machine, program from v$session where sid=&sid;

set line 150
col OWNER for a20
col OBJECT_NAME for a20
select s.SID,s.SERIAL#,lo.PROCESS,lo.LOCKED_MODE,do.OWNER,do.OBJECT_NAME,do.OBJECT_TYPE
  from v$locked_object lo, dba_objects do, v$session s
  where lo.OBJECT_ID=do.OBJECT_ID and lo.SESSION_ID=s.SID;

alter system kill session '&sid,&serial';

select sid,type,lmode,request,ctime from v$lock
  where type in ('TM','TX') order by 1,2;
```

## 生成删除所有普通用户及其数据脚本
``` sql
select 'drop user ' || username || ' cascade;' from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE');
```

## 启动/关闭数据库
``` sql
startup
startup nomount
startup mount
startup open read only;
startup restrict
```

## 查看创建表等数据库对象时的DDL语句
``` sql
desc dbms_metadata
------------------------------------------------------------------------------------------
FUNCTION GET_DDL RETURNS CLOB
 Argument Name                  Type                    In/Out Default?
 ------------------------------ ----------------------- ------ --------
 OBJECT_TYPE                    VARCHAR2                IN
 NAME                           VARCHAR2                IN
 SCHEMA                         VARCHAR2                IN     DEFAULT
 VERSION                        VARCHAR2                IN     DEFAULT
 MODEL                          VARCHAR2                IN     DEFAULT
 TRANSFORM                      VARCHAR2                IN     DEFAULT
------------------------------------------------------------------------------------------

set long 9999
set pagesize 9999
select dbms_metadata.get_ddl('&OBJECT_TYPE','&NAME','&SCHEMA') from dual;

Enter value for object_type: TABLE
Enter value for name: YTHYHDZ
Enter value for schema: BFBHDD9
```


## 实现将SYS用户的操作信息记录到操作系统日志中
``` perl
alter system set audit_syslog_level='USER.NOTICE' scope=spfile;
alter system set audit_sys_operations=TRUE scope=spfile;
alter system set audit_trail=none scope=spfile;
# 调整后的参数生效需重启数据库

# 查看系统日志
tail -100f /var/log/messages
```

## 查找数据库中无效对象
``` sql
col OWNER for a15
col OBJECT_NAME for a50
select OWNER,OBJECT_NAME,OBJECT_TYPE,STATUS from dba_objects 
  where OWNER not in ('SYS','SYSTEM') and STATUS='INVALID';
```

## 统计数据库中各类用户对象的数量
``` sql
select o.owner,o.OBJECT_TYPE,COUNT(*) 
from dba_objects o, 
(select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE','SYSMAN','DBSNMP')) u
where u.username=o.owner group by o.owner,o.OBJECT_TYPE order by 1,2;
```

## 列出数据库中指定类型对象列表
``` sql
col object_name for a50
select owner,object_name,object_type
from dba_objects o,
(select username from dba_users where account_status='OPEN' and username not in ('SYSTEM','SYS','GOLDENGATE','SYSMAN','DBSNMP')) u
where o.owner=u.username and object_type in ('TRIGGER','PROCEDURE','JOB')
order by 1,3,2;
```