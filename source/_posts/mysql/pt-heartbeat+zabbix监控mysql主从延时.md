---
title: pt-heartbeat+zabbix监控mysql主从延时
date: 2018-07-01 22:59:58
tags:  [mysql,mysql维护 ]
categories: [mysql,mysql维护 ]
comments: true
copyright: true
---



## pt-heartbeat+zabbix监控mysql主从延时

### 前言
传统上,大家查看和监控mysql的主从延时都是通过执行show slave status命令来观察Slave_IO_Running和Slave_SQL_Running这两个线程的运行情况.以及Seconds_Behind_Master参数值来判断从库同步是否有延时.  
但是.这些参数其实上并不准确.

笔者在实际工作就遇到这个问题.这个故障也很好复现.当master主库的mysql服务重启后(从库不做任何操作),这个时候从库的所有参数都是正常的.Seconds_Behind_Master参数的值也是0.但是实际上从库没有同步任何数据直到手动执行start slave命令

<!--more-->

幸好,percona提供了一整套的维护mysql的工具箱.percona toolkit.其中就有一款pt-heartbeat工具用来监控主从复制的延时情况.

除此之外,toolkit还包含各种方便我们维护mysql的工具.关于percona的toolkit工具大家可以访问官网详细了解:[percona toolkit](https://www.percona.com/doc/percona-toolkit/LATEST/index.html)

---

### 简介

此章主要介绍pt-heartbeat工具.包含以下几个方面:
- pt-heartbeat介绍
- pt-heartbeat安装
- pt-heartbeat前提条件
- pt-heartbeat使用
- 大并发写入的情况下,延时监控
- 生产上使用pt-heartbeat优化
- zabbix+pt-heartbeat监控主从同步延时

---

### 概述

pt-heartbeat在主库上创建一个heartbeat表,然后定期(默认1s)持续更新时间戳,
然后在从库上读取被更新的时间戳后与本地系统时间对比来得出其延迟。

所以使用heartbeat要满足2个前提:  
1.主从库都需要安装pt-heartbeat工具  
2.主从库的时间要保持一致.否则会出现偏差

关于官网的介绍可以参考:[pt-heartbeat](https://www.percona.com/doc/percona-toolkit/LATEST/pt-heartbeat.html)

---

### 测试环境:

master库: centos7 IP地址:10.0.4.230

slave库: centos7 IP地址:10.0.4.231

---

### pt-heartbeat安装

以下主要介绍redhat/centos的yum方式

1.添加percona的Yum源

```
sudo yum install http://www.percona.com/downloads/percona-release/redhat/0.1-6/percona-release-0.1-6.noarch.rpm
```
2.yum安装

```
sudo yum install percona-toolkit
```
安装完成后可以看到该工具箱包含很多管理工具:

```
[root@server-6 ~]# ll /usr/bin/pt*
-rwxr-xr-x. 1 root root   41754 May 22 01:56 /usr/bin/pt-align
-rwxr-xr-x. 1 root root  269347 May 22 01:56 /usr/bin/pt-archiver
-rwxr-xr-x. 1 root root    3891 Nov  6  2016 /usr/bin/ptaskset
-rwxr-xr-x. 1 root root  170420 May 22 01:56 /usr/bin/pt-config-diff
-rwxr-xr-x. 1 root root  167616 May 22 01:56 /usr/bin/pt-deadlock-logger
-rwxr-xr-x. 1 root root  165964 May 22 01:56 /usr/bin/pt-diskstats
-rwxr-xr-x. 1 root root  170737 May 22 01:56 /usr/bin/pt-duplicate-key-checker
-rwxr-xr-x. 1 root root   50164 May 22 01:56 /usr/bin/pt-fifo-split
-rwxr-xr-x. 1 root root  151447 May 22 01:56 /usr/bin/pt-find
-rwxr-xr-x. 1 root root   67311 May 22 01:56 /usr/bin/pt-fingerprint
-rwxr-xr-x. 1 root root  134593 May 22 01:56 /usr/bin/pt-fk-error-logger
-rwxr-xr-x. 1 root root  223097 May 22 01:56 /usr/bin/pt-heartbeat
-rwxr-xr-x. 1 root root  227851 May 22 01:56 /usr/bin/pt-index-usage
-rwxr-xr-x. 1 root root   32412 May 22 01:56 /usr/bin/pt-ioprofile
-rwxr-xr-x. 1 root root  255247 May 22 01:56 /usr/bin/pt-kill
-rwxr-xr-x. 1 root root   21920 May 22 01:56 /usr/bin/pt-mext
-rwxr-xr-x. 1 root root 6691168 May 22 01:56 /usr/bin/pt-mongodb-query-digest
-rwxr-xr-x. 1 root root 7168736 May 22 01:56 /usr/bin/pt-mongodb-summary
-rwxr-xr-x. 1 root root  108030 May 22 01:56 /usr/bin/pt-mysql-summary
-rwxr-xr-x. 1 root root  419918 May 22 01:56 /usr/bin/pt-online-schema-change
-rwxr-xr-x. 1 root root   24661 May 22 01:56 /usr/bin/pt-pmp
-rwxr-xr-x. 1 root root  526640 May 22 01:56 /usr/bin/pt-query-digest
-rwxr-xr-x. 1 root root 4373504 May 22 01:56 /usr/bin/pt-secure-collect
-rwxr-xr-x. 1 root root   78095 May 22 01:56 /usr/bin/pt-show-grants
-rwxr-xr-x. 1 root root   37791 May 22 01:56 /usr/bin/pt-sift
-rwxr-xr-x. 1 root root  146590 May 22 01:56 /usr/bin/pt-slave-delay
-rwxr-xr-x. 1 root root  131383 May 22 01:56 /usr/bin/pt-slave-find
-rwxr-xr-x. 1 root root  184554 May 22 01:56 /usr/bin/pt-slave-restart
-rwxr-xr-x. 1 root root   74606 May 22 01:56 /usr/bin/pt-stalk
-rwxr-xr-x. 1 root root   90823 May 22 01:56 /usr/bin/pt-summary
-rwxr-xr-x. 1 root root  454981 May 22 01:56 /usr/bin/pt-table-checksum
-rwxr-xr-x. 1 root root  404729 May 22 01:56 /usr/bin/pt-table-sync
-rwxr-xr-x. 1 root root  247750 May 22 01:56 /usr/bin/pt-table-usage
-rwxr-xr-x. 1 root root  332574 May 22 01:56 /usr/bin/pt-upgrade
-rwxr-xr-x. 1 root root  178057 May 22 01:56 /usr/bin/pt-variable-advisor
-rwxr-xr-x. 1 root root  102552 May 22 01:56 /usr/bin/pt-visual-explain
-rwxr-xr-x. 1 root root   66648 Nov  6  2016 /usr/bin/ptx
```
---
### 使用pt-heartbeat的前提条件



**1.保持时间始终同步**

heartbeat的原理是通过接收master更新过来的时间戳,然后对比本地系统当前时间判断时间延时.这就意味着slave库和master库的时间要保持高度的一致,否则如果时间不一致,则pt-heartbeat采集到的延时数据没有任何用处,甚至会产生误导,导致意外后果.  

可以通过ntpdate命令来同步网络时钟.

```
ntpdate -u ntp.api.bz
```

为了时钟保持时间同步.在每台服务器上通过crontab定时任务,定期同步时间.例如每隔30分钟同步一次:

```
vim /etc/crontab
 */30 * *  *  *   root  /sbin/ntpdate -u ntp.api.bz
```
另外,为了防止服务器重启后时间不一致,还需要让此命令开机自动执行.例如在centos7系统下可以如下配置:

```
chmod +x /etc/rc.d/rc.local
echo "/sbin/ntpdate -u ntp.api.bz" >>  /etc/rc.d/rc.local
```
> Note:理解和注意master,slave服务器的时间同步极其重要.否则会导致出现意外的误判,甚至导致严重故障



**2.配置主从复制环境**

pt-heartbeat工作基于主从复制,所以前提需要搭建好主从环境,并且确保他们能正常工作.master和slave数据已经同步一致.否则使用pt-heartbeat将毫无意义. 

关于主从复制环境配置在此不做任何介绍,有兴趣可以翻阅我的另外一篇笔记:mysql5.7使用GTID配置主从复制

---

### pt-heartbeat使用

以下摘取一些主要的选项:

```
Options:

  --ask-pass                  在交互界面输入数据库密码
  --charset=s             -A  默认字符集
  --check                     检查从库的延时,只检查一次,检查完就退出
  --create-table              如果heartbeat表不存在.则主库创建heartbeat表
  --daemonize                 后台挂起运行
  --database=s            -D  需要监控延时的数据库
  --file=s                    如果使用monitor选项,则输出结果到一个文件中
  --help                      查看帮助信息
  --host=s                -h  连接数据库的IP地址
  --interval=f                多久一次去更新主库的时间戳,以及从库多久一次去查看接收到的时间戳
  --log=s                     当daemonize模式运行时.打印输出到一个日志文件
  --master-server-id=s        从库指定master库的server-id
  --monitor                   持续监控延时情况.这里和check选项的监控一次就退出相反
  --password=s            -p  在命令行上明文指定密码
  --port=i                -P  数据库端口,默认3306
  --replace                   使用replace参数替换update参数
  --skew=f                    多长时间的延时偏量(默认0.5秒).也就是超过0.5秒不认为有延时
  --slave-password=s          连接slave库的密码
  --slave-user=s              连接slave库的用户名
  --socket=s              -S  连接数据库的socket
  --stop                      停止运行该工具（--daemonize），在/tmp/目录下创建一个“pt-heartbeat-sentinel” 文件。后面想重新开启则需要把该临时文件删除，才能开启（--daemonize）。
  --table=s                   创建一个表用来更新时间戳(默认是heartbeat)
  --update                    更新master库的时间戳
```

> note.如果安装后执行pt-heartbeat报如下错误:

```
install_driver(mysql) failed: Attempt to reload DBD/mysql.pm aborted.
Compilation failed in require at (eval 31) line 3, <STDIN> line 1. 
at /usr/bin/pt-heartbeat line 2888.
```

这可能是缺少了mysql的lib库.执行如下操作:

    ln -s /usr/lib64/mysql/libmysqlclient.so.18 /lib64/

---



**step1:** 在Master主库上以守护进程方式运行heartbeat

```
[root@localhost ~]$pt-heartbeat --user=root --ask-pass --host=127.0.0.1  --create-table -D dwd  --update --replace --daemonize
Enter password:
[root@localhost ~]$
```

查看进程:

```
[root@localhost ~]$ps aux | grep pt-heartbeat | grep -v grep
root       8057  0.0  1.5 226420 16016 ?        Ss   12:31   0:00 perl /usr/bin/pt-heartbeat --user=root --ask-pass --host=127.0.0.1 --create-table -D dwd --update --replace --daemonize
```

查看被创建的heartbeat表:

```
mysql> desc heartbeat;
+-----------------------+---------------------+------+-----+---------+-------+
| Field                 | Type                | Null | Key | Default | Extra |
+-----------------------+---------------------+------+-----+---------+-------+
| ts                    | varchar(26)         | NO   |     | NULL    |       |
| server_id             | int(10) unsigned    | NO   | PRI | NULL    |       |
| file                  | varchar(255)        | YES  |     | NULL    |       |
| position              | bigint(20) unsigned | YES  |     | NULL    |       |
| relay_master_log_file | varchar(255)        | YES  |     | NULL    |       |
| exec_master_log_pos   | bigint(20) unsigned | YES  |     | NULL    |       |
+-----------------------+---------------------+------+-----+---------+-------+
6 rows in set (0.09 sec)
```

查看时间戳更新情况.可以看到每秒更新一次:

```
mysql> select * from heartbeat;
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| ts                         | server_id | file             | position  | relay_master_log_file | exec_master_log_pos |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| 2018-06-22T12:32:56.004830 |       230 | mysql-bin.000007 | 305862718 | NULL                  | NULL                |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
1 row in set (0.09 sec)

mysql> select * from heartbeat;
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| ts                         | server_id | file             | position  | relay_master_log_file | exec_master_log_pos |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| 2018-06-22T12:32:57.001820 |       230 | mysql-bin.000007 | 305863097 | NULL                  | NULL                |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
1 row in set (0.09 sec)
```

**step2:** 从库持续监控延时情况

```
[root@localhost ~]$pt-heartbeat --user=root --password=xxxx -D dwd --master-server-id=230 --monitor
0.00s [  0.00s,  0.00s,  0.00s ]
0.00s [  0.00s,  0.00s,  0.00s ]
0.00s [  0.00s,  0.00s,  0.00s ]
0.00s [  0.00s,  0.00s,  0.00s ]
0.00s [  0.00s,  0.00s,  0.00s ]
0.00s [  0.00s,  0.00s,  0.00s ]
0.00s [  0.00s,  0.00s,  0.00s ]
0.00s [  0.00s,  0.00s,  0.00s ]
```

> monitor选项表示持续监控,并且打印结果. 如果加上--print-master-server-id还可以打印主库的server-id
>
> 上述的四个值代表:实时延时[过去1分钟延时,5分钟延时,15分钟延时]

**step3** 利用check选项,监控一次就退出

```
[root@localhost ~]$pt-heartbeat --user=root --password=xxxx -D dwd --master-server-id=230 --check
0.00
[root@localhost ~]$
```

主要的使用方法就到这.

---

### 模拟大并发写入.pt-heartbeat监控延时情况

**step1**.master库创建一个表

```
mysql> CREATE TABLE `t_user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `age` tinyint(4) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  `update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.01 sec)
```

**step2**.执行一个存储过程,模拟写入百万条数据

```
mysql> delimiter $$
DROP PROCEDURE IF EXISTS proc_batch_insert;
CREATE PROCEDURE proc_batch_insert()
BEGIN
DECLARE pre_name BIGINT;
DECLARE ageVal INT;
DECLARE i INT;
SET pre_name=187635267;
SET ageVal=100;
SET i=1;
WHILE i < 1000000 DO
        INSERT INTO t_user(`name`,age,create_time,update_time) VALUES(CONCAT(pre_name,'@qq.com'),(ageVal+1)%30,NOW(),NOW());
SET pre_name=pre_name+100;
SET i=i+1;
END WHILE;
END $$
 
delimiter ;
call proc_batch_insert();
```

**step3:**延时明显升高

```
6.00s [  6.00s,  3.30s,  1.10s ]
6.00s [  6.00s,  3.32s,  1.11s ]
6.00s [  6.00s,  3.34s,  1.11s ]
6.00s [  6.00s,  3.36s,  1.12s ]
6.00s [  6.00s,  3.38s,  1.13s ]
6.01s [  6.00s,  3.40s,  1.13s ]
6.00s [  6.00s,  3.42s,  1.14s ]
6.01s [  6.00s,  3.44s,  1.15s ]
```
---

### 生产中使用pt-heartbeat的优化

1.在从库新建一个专用于检查延时的mysql账户,且只赋予heartbeat表的select权限.例如下面创建一个heartbeat账户:

```
mysql> create user 'heartbeat'@'localhost' identified by '密码';
Query OK, 0 rows affected (0.33 sec)
mysql> grant select on 数据库.heartbeat to 'heartbeat'@'localhost';
Query OK, 0 rows affected (0.01 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.13 sec)

mysql> 
```



2.默认情况下,master库每隔1秒更新时间戳.这可能会过于频繁.使用--interval参数可以设置master库的更新时间戳频率.例如,master每隔10s更新一次时间戳

```
[root@localhost ~]$/usr/bin/pt-heartbeat --user=root --ask-pass --host=127.0.0.1 --create-table -D dwd --interval=10 --update --replace --daemonize
```

检查heartbeat库,发现确实是10秒才更新一次:

```
mysql> select * from heartbeat;
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| ts                         | server_id | file             | position  | relay_master_log_file | exec_master_log_pos |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| 2018-06-22T14:46:35.003090 |       230 | mysql-bin.000007 | 308874907 | NULL                  | NULL                |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
1 row in set (0.09 sec)

mysql> select * from heartbeat;
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| ts                         | server_id | file             | position  | relay_master_log_file | exec_master_log_pos |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
| 2018-06-22T14:46:45.001820 |       230 | mysql-bin.000007 | 308875286 | NULL                  | NULL                |
+----------------------------+-----------+------------------+-----------+-----------------------+---------------------+
1 row in set (0.08 sec)
```

查看slave服务器的延时.由于slave默认每秒检查所以延时会一直持续,直到第10秒时候接收到master更新.

```
0.00s [  0.00s,  0.00s,  0.00s ]
1.00s [  0.02s,  0.00s,  0.00s ]
2.00s [  0.05s,  0.01s,  0.00s ]
3.00s [  0.10s,  0.02s,  0.01s ]
4.00s [  0.17s,  0.03s,  0.01s ]
5.00s [  0.25s,  0.05s,  0.02s ]
6.00s [  0.35s,  0.07s,  0.02s ]
7.00s [  0.47s,  0.09s,  0.03s ]
8.00s [  0.60s,  0.12s,  0.04s ]
9.00s [  0.75s,  0.15s,  0.05s ]
0.00s [  0.75s,  0.15s,  0.05s ]
1.00s [  0.77s,  0.15s,  0.05s ]
```

---

### pt-heartbeat+zabbix 监控延时

工作中可以通过shell脚本来持续监控heartbeat,判断主从是否有延时.方式主要有两种

1.编写shell脚本获取heartbeat采集到的数据,并且通过crontab定期执行脚本.利用邮件方式报警延时情况

2.通过zabbix监控shell脚本采集heartbeat数据.通过邮件,微信,钉钉等报警方式通告延时情况

这里我们选择了zabbix,因为zabbix可以实时观察延时情况,而且可以分不通的报警等级.报警到工作的钉钉群

> note:
>
> 1.以下的配置大部分都在slave服务器,且该服务器上安装了zabbix agent
>
> 2.我们使用的是zabbix 3.4版本.可能并不是每个步骤都适用于其他版本



接下来配置zabbix监控项

1.编写一个shell脚本.采集heartbeat的延时数据.使用--check选项.

```
vim mysql-heartbeat.sh
#!/bin/bash

#description: use percona pt-heartbeat to monitor the delay of MASTER-SLAVE replication

delay=$(/usr/bin/pt-heartbeat --host=127.0.0.1 --user=heartbeat --password=密码 -D 数据库  --master-server-id=MASTER库的server-ID  --check)
echo ${delay%.*}
```

2.将该脚本移动到/etc/zabbix/script目录下(这个目录需要手动创建),且赋予执行权限.出于安全考虑.设置为只有zabbix用户可以查看

```
[root@server-6 script]# ll /etc/zabbix/script/
-rwx--x--x. 1 zabbix zabbix 263 Jun 25 17:22 mysql-heartbeat.sh
```

> 当然,你也可以将该脚本放在任何您想存放的地方

3.在/etc/zabbix/zabbix_agentd.d目录下编写UserParameter文件

```
[root@server-6 script]# cd /etc/zabbix/zabbix_agentd.d/

[root@server-6 zabbix_agentd.d]# vim  userparameter_heartbeat.conf

UserParameter=heartbeat.delay,/etc/zabbix/script/mysql-heartbeat.sh
```

> note: 这里的UserParameter为固定格式.
>
> heartbeat.delay表示zabbix的监控项
>
> 后面的脚本表示监控项获取到的数据.也就是说zabbix检查heartbeat.delay监控项的数据,其实就是执行后面的shell脚本,并得到脚本的返回结果

4.配置zabbix agentd配置文件.修改include字段,让zabbix agent读取UserParameter配置文件:

```
[root@server-6 zabbix_agentd.d]# vim /etc/zabbix/zabbix_agentd.conf
Include=/etc/zabbix/zabbix_agentd.d/*.conf
```

5.重启zabbix_agentd服务

```
pkill zabbix_agentd
zabbix_agentd -c /etc/zabbix/zabbix_agentd.conf
```

6.在zabbix server服务器检查是否可以正常获取到监控项的数据

```
[root@server-4 ~]# zabbix_get -s 10.0.0.216 -k heartbeat.delay
2
```

> -s 参数表示zabbix agent的IP地址
>
> -k 参数表示监控项
>
> 这里可以正常获取到监控项的数据,也就是shell脚本的执行结果

---



既然zabbix server通过命令行可以获取到监控项的数据,接下来只要配置zabbix的web界面就可以了.

1.在slave库配置监控项

![](https://img1.jesse.top/mysql-heartbeat.png)



2.配置图形

![](https://img1.jesse.top/mysql-heartbeat1.png)



3.配置触发器

![](https://img1.jesse.top/mysql-heartbeat2.png)

> note: 为了防止因为网络抖动造成的频繁的延时报警.这里表达式设置为连续3次采集的数据都大于10秒才报警.也可以换成另外一种表达式:
>
> {server-6:heartbeat.delay.last(0)}>30
>
> 这个表达式表示监控项的最新数据只要大于30秒就报警.大于30秒就意味着连续3次报警都延时.我认为这种效果更好

另外.再设置一档报警.比如5分钟延时报警,此时报警等级为严重.

![](https://img1.jesse.top/mysql-heartbeat3.png)



4.查看监控数据

![](https://img1.jesse.top/mysql-heartbeat4.png)



5.测试触发器是否工作正常

![](https://img1.jesse.top/mysql-heartbeat5.png)



---

至此.成功利用zabbix来监控mysql的主从复制同步情况.

```
install_driver(mysql) failed: Attempt to reload DBD/mysql.pm aborted.
Compilation failed in require at (eval 31) line 3, <STDIN> line 1.

 at /usr/bin/pt-heartbeat line 2888.
```

```
ln -s /usr/lib64/mysql/libmysqlclient.so.18 /lib64/
```






