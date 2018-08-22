---
title: mysql 5.7配置binlog主从复制
date: 2018-07-04 22:59:58
tags:  [mysql,mysql维护  ]
categories: [mysql,mysql维护 ]
comments: true
copyright: true
---



### mysql 5.7配置binlog主从复制



#### 一.本教程配置时间

时间:2018-05-08

#### 二.环境:

master数据库:

        hostname:tongji-db
    
        内网IP:xx.xx.xx.xx

<!--more-->

 slave数据库:

        hostname:tongji-1
    
        内网IP:xx.xx.xx.xx

两台数据库服务器的操作系统都是CentOS 7.4

mysql版本都是5.7



#### 三.背景:

1.tongji-db为master库,并且已经开启了bin_log日志,而且也有pos信息:

```
mysql> show master status;
+------------------+-----------+--------------+------------------+-------------------+
| File             | Position  | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+-----------+--------------+------------------+-------------------+
| mysql-bin.003321 | 446544409 |              |                  |                   |
+------------------+-----------+--------------+------------------+-------------------+
1 row in set (0.06 sec)
```



2.tongji-1是新slave.而且是新数据库(之前的tongji-1上的数据库已经迁移到tongji-db)



#### 四.配置步骤:

**一.使用xtrabackup备份master数据库**

1.使用xtrabackup对tongji-db的master数据库进行备份,最好是全备.

这个步骤在这里不讨论,可以查看之前编制过的其他文档



2.将master的数据库备份文件拷贝到tongji-1这台slave服务器



**二.在Master数据库创建一个用户,用于slave连接进行主从复制**

```
mysql> create user 'repl'@'%' identified by '密码';
Query OK, 0 rows affected
mysql> grant replication slave on . to 'repl'@'%';
Query OK, 0 rows affected
```

**三.在slave服务器上安装xtrabackup**

**1.安装xtrabackup**

```
yum install http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
yum install percona-xtrabackup-24

[root@tongji-1 data]# which xtrabackup
/bin/xtrabackup
```



**2.由于master的备份文件是压缩备份的.所以需要先解压缩,**

解压缩需要用到qpress软件.这个软件也在上文的xtrabackup的percona软件仓库里

```
yum install qpress
```



**四.,利用xtrabackup将master备份文件恢复到slave服务器**

**1.开启一个screen进行数据库还原**

```
[root@tongji-1 data]# screen -S xtrabackup
```

**2.解压master的压缩备份文件**./data/backups/19/base/这个目录是master复制过来的备份文件

```
[root@tongji-1 data]# xtrabackup --decompress --target-dir=/data/backups/19/base/

解压成功会出现completed ok!
180507 22:47:28 [01] decompressing ./dwd_webcron/cron_host.ibd.qp
180507 22:47:29 [01] decompressing ./dwd_webcron/cron_task_log.frm.qp
180507 22:47:29 [01] decompressing ./dwd_webcron/cron_task_host.ibd.qp
180507 22:47:29 completed OK!

解压缩后发现数据库目录文件容量大幅增加
[root@tongji-1 data]# du -sh backups/19/base/
646G    backups/19/base/
```

**3.修改备份目录的权限**

```
chown -R mysql.mysql  backups/19/base/
```

**4.prepare还原工作**

```
[root@tongji-1 data]# xtrabackup --user='root' --password='密码' --prepare --target-dir=/data/backups/19/base/

prepare成功,会出现completed ok
InnoDB: Shutdown completed; log sequence number 2385286774312
180507 23:07:50 completed OK
```

**5.开始还原数据库.**

这里使用了move-back参数而没有使用copy-back.是因为考虑到磁盘容量不够,所以还原时删除原始备份文件

```
xtrabackup --user='root' --password='密码' --move-back --target-dir=/data/backups/19/base/ --datadir=/data/mysql

还原成功,会出现completed ok
180507 23:11:29 [01] Moving ./ib_buffer_pool to /data/mysql/ib_buffer_pool
180507 23:11:29 [01]        ...done
180507 23:11:29 completed OK!
```



**6.修改数据库的数据文件目录权限**

```
chown -R mysql:mysql mysql
```

---



#### 五.配置slave

**1.查看截止到备份时,master的binlog日志信息**

slave配置复制时,需要指明从master的哪个binlog日志,哪个位置开始复制同步.所幸xtrabackup备份的时候已经备份了master的Bin_log日志和POS信息..

在slave上可以进入备份文件目录查看这些信息.

```
cd /data/backups/19/base
[root@tongji-1 base]# cat xtrabackup_binlog_info
mysql-bin.002537    757197558
```

**xtrabackup_binlog_info**文件记录了master备份时候的binlog日志信息.slave还原数据库后,从这里开始同步master数据库.

>  注意:slave还原数据库后,需要从这里开始同步master数据库.而不是从master上最新的bin日志位置同步.**这一点至关重要**



**2.配置数据库配置文件中的slave配置.**

slave服务器的mysql配置文件在/etc/percona-server.conf.d目录下

```
vim /etc/percona-server.conf.d/mysqld.cnf

添加下面2个参数
server_id=179  #这个是slave的server_id,这个ID值和master不能重复,这里我取IP地址的最后一位
read_only=1    #这个表示slave数据库仅仅只读
```



**此外还有下列参数可选:**

replicate-do-db =   #后面跟数据库名.表示只复制某个数据库

replicate-ignore-db  #设定需要忽略的复制数据库 （多数据库使用逗号，隔开）

replicate-do-table  #设定需要复制的表

replicate-ignore-table #设定需要忽略的复制表 

replicate-wild-do-table  #同replication-do-table功能一样，但是可以通配符

replicate-wild-ignore-table  #同replication-ignore-table功能一样，但是可以加通配符



> 注意.如果是要主从同步多个指定的数据库,需要指定多个replicate-do-db语句.这是[官网说明](https://dev.mysql.com/doc/refman/8.0/en/replication-options-slave.html#option_mysqld_replicate-do-db)



**3.启动mysql服务**

```
systemctl start mysqld.service
```

**4.进入mysql命令行,配置master信息**

```
mysql -h127.0.0.1 -uroot -p

mysql> change master to
   -> MASTER_HOST='xx.xx.xx.xx',          #master数据库的IP地址
  -> MASTER_USER='repl',                  #master上创建的用户
  -> MASTER_PASSWORD='密码',   #该用户密码
  -> MASTER_LOG_FILE='mysql-bin.002537',  #slave上还原时的master数据库bin日志信息.注意这个bin日志信息是master备份数据库时的bin信息,不是最新的.
   -> MASTER_LOG_POS=757197558;            #slave还原时,记录的master数据库的Position记录
Query OK, 0 rows affected, 1 warning (0.04 sec)
```

**5.启动slave**

```
mysql> START SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

**6.查看slave同步状态**

通过以下命令可以查看slave的同步状态信息.可以看到master的信息,以及正在同步的master_log_file的bin文件名,和pos位置信息

另外要确保slave_IO_Running和slave_sql_running这两个线程正常工作.

```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: xx.xx.xx.xx
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.003319
          Read_Master_Log_Pos: 906123896
               Relay_Log_File: tongji-1-relay-bin.002165
                Relay_Log_Pos: 322133185
        Relay_Master_Log_File: mysql-bin.003259
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
          Exec_Master_Log_Pos: 322132972
              Relay_Log_Space: 65330834662
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 612204
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 63306
                  Master_UUID: bb0a31bf-ef6f-11e7-bdac-00163e0c8952
             Master_Info_File: /data/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Reading event from the relay log
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
1 row in set (0.00 sec)
```

> 对以上命令输出的部分解析:

**Master_Log_File**: mysql-bin.003319  #当前master的最新binlog日志

**Relay_Master_Log_File**: 目前读取到的master的binlog日志

**Relay_Log_File**: 从库记录的binlog日志

**Slave_IO_Running**: IO线程工作情况

**Slave_SQL_Running**:SQL线程工作情况

**Seconds_Behind_Master**:slave落后master时间,单位秒..此值仅供参考

**Master_UUID**:此值使用GTID复制时有效

