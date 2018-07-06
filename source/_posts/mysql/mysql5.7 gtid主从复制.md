---
title: mysql5.7 GTID主从复制
date: 2018-06-23 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---

## GTID主从复制

#### 简介

mysql5.7开始支持两种主从复制:

* 基于传统的binlog和position复制
* 基于GTID复制

这里主要介绍GTID复制的配置.

有关GTID复制的原理请参考官方文档:[GTID原理介绍](https://dev.mysql.com/doc/refman/5.7/en/replication-gtids.html)

<!--more-->

------------

#### GTID概述

这里主要介绍几点GTID的概念:

- GTID(global transaction identifieds) 全局事务标识
- GTID是全局唯一性的,每一个事务对应一个GTID
- 一个GTID在服务器上只执行一次.任何尝试用相同的GTID执行的动作都会被忽略
- GTID用来代替传统的复制方法，不在使用binlog+pos开启复制。而是使用master_auto_postion=1的方式自动匹配GTID断点进行复制。

**GIID的组成部分**

GTID由两部分组成:

UUID+SN号

UUID:每个mysql实例拥有一个唯一的ID.
SN号.mysql服务以数字1开始递增,执行一次事务SN号加1.一个事务对应一个数值.

---

#### GITD主从复制配置步骤

以下教程参考官方文档:[Setting Up Replication Using GTIDs](https://dev.mysql.com/doc/refman/5.7/en/replication-gtids-howto.html)

##### 主要包含以下步骤:

1.如果之前已经配置了主从复制,通过设置双方服务器为只读来同步双方服务器  
2.停止双方服务  
3.配置GTID,重新启动服务  
4.配置slave从库上的master信息  
5.开始一个新的备份,GTID开启后,没有包含GTID的Binlog不能使用.所以在使用新的配置之前进行备份.这意味着需要对数据库进行一次备份  
6.开启slave.  
7.在双方服务器上关闭read-only模式

---

#### 详细步骤

接下来的例子中,假设两台服务器已经配置了基于binlog和position主从复制的master和slave.

如果你是两台新的服务器.需要先做如下2步配置:

1.创建用户组从复制的mysql账号:

```
mysql> CREATE USER 'repl'@'%.example.com' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%.example.com';
```

2.配置my.cnf配置文件.指定binlog日志和server-id

```
[mysqld]
log-bin=mysql-bin
server-id=1
```

##### 详细步骤开始

> note: 接下来的大部分步骤都需要mysql的root账户,或者其他有SUPER权限的账户.

**step1.同步master和slave两台服务器.** 

这个步骤是指使用Binlog和position来同步的服务器,如果是新的服务器,从步骤3开始执行.在每台服务器上设置read_ony系统变量为ON,确保服务器只读,

```
mysql> SET @@global.read_only = ON;
```

等待所有的正在执行的事务提交或者回滚,然后允许Slave和master保持一致.**在继续接下来步骤之前确保slave完成所有Updates,这极其重要**

> Important:  
> 当GTID特性开启后,没有GTIDs的事务日志将不能用于服务器,理解这一点至关重要.在继续接下来操作之前,一定要确保未包含GTIDs的事务日志在整个mysql拓扑中已经不存在.

**step2.在每台服务器上关闭mysql服务.**

使用mysqladmin工具关闭mysql:

```
shell> mysqladmin -uusername -p shutdown
```

**step3.配置GTID,开启mysql服务**

设置gitd_mode变量为ON,开启GTID模式.enfore_gtid_consistency变量确保只有基于GTID的安全语句才会被记录.另外,你还应该在SLAVE服务器上使用--skip-slave-start选项在配置slave之前启动mysql.

> 关于GTID的更多选项和变量,可以参考 [Global Transaction ID Options and Variables](https://dev.mysql.com/doc/refman/5.7/en/replication-options-gtids.html)

在mysql5.7.5中因为有了额外的mysql.gtid_executed表,所以不强制要求开启二进制日志来使用GTID.这就意味着可以在slave服务器上没有二进制日志的情况下使用GTID.但是Master服务器必须要开启二进制日志去同步.

配置文件例子:

在Master服务器上:

```
log-bin=mysql-bin
binlog_format = row
server-id=230
expire_logs_days = 7

#GTID:
gtid_mode=ON
enforce_gtid_consistency=true
```

在Slave服务器上:

```
server-id=231
gtid_mode=ON
enforce_gtid_consistency=true
#log-slave-updates = ON
skip_slave_start=1
read_only=1
```

> 很多教程在MASTER服务器开启了log_slave_updates参数.经过官网查看[log_slave_updates](https://dev.mysql.com/doc/refman/8.0/en/replication-options-slave.html)选项是指slave写入从Master服务器接收到的updates.并且slave的SQL线程在slave自己的二进制日志中执行.默认情况下log-bin选项(控制二进制日志的选项)是开启的.二进制日志必须在从库上开启才能记录Updates.  
除非指定了skip-log-bin关闭二进制记录,否则log-slave-update默认是开启的.如果在二进制日志开启的情况下,你需要关闭slave的update日志,可以使用skip-log-slave-updates.  

>log-slave-updates开启了链式复制.例如:
```
A -> B -> C
```
> A服务器是SLAVE B服务器的MASTER.而B服务器又是slave C的MASTER.在这种情况下B既是Mster又是slave.这个时候就需要开启二进制日志和log-slave-update选项.此时,从A接收到的updates会被B记录到自己的二进制日志,然后传递给C

>所以这里我个人认为.在Master服务器上不需要开启这个选项,而且如果slave从库不需要传递update到其他从库的话,也不需要开启这个选项.


**step4.配置slave使用基于GTID的自动寻址.**

这里有点类似于二进制复制的配置:

```
mysql> CHANGE MASTER TO
     >     MASTER_HOST = host,
     >     MASTER_PORT = port,
     >     MASTER_USER = user,
     >     MASTER_PASSWORD = password,
     >     MASTER_AUTO_POSITION = 1;
```

传统二进制复制的Master_log_file选项或者MASTER_log_pos选项不能和MASTER_AUTO_POSITION = 1选项共存.这会导致Change MASTER TO选项错误.这就意味着,只能使用二进制复制或者GTID复制其中的一种

**step5.启动一个新的backup**

在开启GTIDs特性之前所做的backups已经不能在任何开启GTID的mysql服务器上使用了.开启一个新的备份,这样你的服务器就不会出于无备份状态了.

例如.你可以在服务器上执行flush logs,然后显示的执行一个备份,或者等待周期性的自动备份.

**step6.开启slave,并且关闭read-only模式**

开启slave:

```
mysql> START SLAVE;
```

如果你的slave从库是只读的,那么关闭read-only模式不是必须的.为了允许服务器开始接受更新,需要关闭只读模式

```
mysql> SET @@global.read_only = OFF;
```

基于GTID的复制应该可以正常运行了.

---

#### GTID复制的实验


在配置完上述步骤后.进行一个GTID复制的实验,验证GTID主从复制的有效性

> note:在开始之前,请确保master和slave服务器完全同步,而且按照上述步骤开启了GTID的配置


1.在测试数据库中创建个新表.

```
mysql> create table testing(id int(6));
Query OK, 0 rows affected (0.02 sec)
```

2.查看master状态

```
mysql> show master status;
+------------------+----------+--------------+------------------+------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+------------------+----------+--------------+------------------+------------------------------------------+
| mysql-bin.000002 |     1425 |              |                  | d5556272-3cca-11e8-b751-000c29dcb1c7:1-6 |
+------------------+----------+--------------+------------------+------------------------------------------+
1 row in set (0.00 sec)
```
>NOTE:Executed_Gtid_Set栏的值表示一个GTID号码.这个号码由2部分组成.冒号:前面一长串是UUID.1-6表示这是第6个GTID事务,此值的初始值是1-1.每执行一个事务,就加1.

3.在testing表中插入一个值

```
mysql> insert into testing values(1);
Query OK, 1 row affected (0.01 sec)
```

4.再次查看master状态

```
mysql> show master status;
+------------------+----------+--------------+------------------+------------------------------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                        |
+------------------+----------+--------------+------------------+------------------------------------------+
| mysql-bin.000002 |     1681 |              |                  | d5556272-3cca-11e8-b751-000c29dcb1c7:1-7 |
+------------------+----------+--------------+------------------+------------------------------------------+
1 row in set (0.00 sec)

```

> GTID值在上一个事务的基础上加1.变成了7

5.查看slave库的状态

```
mysql> show slave status \G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.0.4.230
                  Master_User: root
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000002
          Read_Master_Log_Pos: 1681
               Relay_Log_File: localhost-relay-bin.000005
                Relay_Log_Pos: 1384
        Relay_Master_Log_File: mysql-bin.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1681
              Relay_Log_Space: 2073
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 230
                  Master_UUID: d5556272-3cca-11e8-b751-000c29dcb1c7
             Master_Info_File: /data/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set: d5556272-3cca-11e8-b751-000c29dcb1c7:1-7
            Executed_Gtid_Set: d5556272-3cca-11e8-b751-000c29dcb1c7:1-7
                Auto_Position: 1
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

>主要关注Slave_IO_Running和Slave_SQL_Running.并且已经看到Master_UUID值和Executed_Gtid_Set参数的master传递过来的GTID号码.这里显示的GTID号码和master一致.

查看从库上testing表:

```
mysql> use dwd;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

mysql> select * from testing;
+------+
| id   |
+------+
|    1 |
+------+
1 row in set (0.00 sec)

```

至此GTID复制正常运行

---

