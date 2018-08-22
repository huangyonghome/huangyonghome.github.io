---
title: mysql备份工具介绍
date: 2018-06-23 22:59:58
tags:  [mysql,mysql维护 ]
categories: [mysql,mysql维护 ]
comments: true
copyright: true
---

## mysql备份工具介绍



本章主要介绍mysql常见的3种的备份(导入,导出)工具.

- mysqldump
- mydumper&&myloader
- xtrabackup

<!--more-->

----

#### 一.mysqldump

**概述:**

mysqldump是MYSQL自带的备份工具.它基于逻辑备份方式产生一套可以被重新执行的原始数据库DDL和DML语句.mysqldump导出一个或者多个mysql数据库,然后复制或者还原到另外一台SQL服务器.

有关更相信的官方文档介绍和参数介绍请查看官网.

---

**用法:**

mysqldump的用法非常简单.以下只介绍几种最常见的用法.更多用法请参考官网.



- **备份数据库**

1.备份数据库

```
mysqldump -uroot -pPassword [database name] > [dump file]
```

 上述命令将指定数据库备份到某dump文件（转储文件）中，比如:

```
mysqldump -uroot -p123 test > test.dump
```



2.跨主机备份.将备份结果直接还原到目的主机.例如:

```
mysqldump --host=host1 --opt sourceDb| mysql --host=host2 -C targetDb
```

>     -C指示主机间的数据传输使用数据压缩 



3.只备份表结构.需要加上--no-data参数

```
mysqldump -uroot -p123 --no-data mydatabase > test.dump
```

---

- **恢复数据库**

1.从文件恢复

```
mysqldump -uuser -ppassword [database name] < [backup file name]
```

> note:  
>
> 1.要恢复的数据库名必须事先创建.  
>
> 2.在恢复时最好加上--database参数,明确指定恢复到哪个数据库.:
>
> ```
> mysqldump -uuser -ppassword --database [database name] < [backup file name]
> ```
>
> 虽然大部分情况下--database参数可有可无,但是有些时候可能会报错



2.source恢复

source恢复方法:

1.登录mysql命令行

2.source /path-to-backup file name

---



### 二.mydumper

----

#### 概述

mydumper是第三方备份工具.详细介绍可以参考官网:[mydumper官网](https://launchpad.net/mydumper)

mydumper对于mysqldump来说最明显的区别就是mydumper支持多线程导入和导出(默认4个线程).所以速度会比mysqldump.

----

#### 用法

**一.备份基本用法**

备份某个数据库

```
mydumper -u用户名 -p密码 -h主机 -P 端口 -B 数据库 -o 备份目的目录
```

备份所有数据库:

```
mydumper u用户名 -p密码 -h主机 -P 端口  -o 备份目的目录
```

备份某个表,如果是多个表,则表名之间用逗号分隔

```
mydumper u用户名 -p密码 -h主机 -P 端口  -B 数据库 -T 该数据库的表名 -o 备份目的目录
```

> 此外mydumper还有其他的可选参数.例如
>
> -c :压缩导出文件
>
> --no-data:仅仅导出表结构
>
> 更多详细参数可以查看官网,或者执行 ./mydumper --help查看

 

**二.还原基本用法**

mydumper备份文件,使用myloader命令还原

用法:

```
myloader -u用户名 -p密码 -h主机 -P 端口 -B 数据库 -d 备份文件所在的目录
```

---



**三.mydumper用法示例**

1.导出test数据库:

```
[root@mysql mydumper-0.9.1]# ./mydumper -u root -p 123456  -B test -o /data/test
```

> note: 
>
> 1.mydumper一定要在命令行显示输入密码..mydumper不像mydump可以执行命令后再在交互界面输入密码 .如果这里参数-p 留空,没有输入密码.则不会提示你输入密码,并且备份所有数据库
>
> 2.mydumper导出的是一个目录,而不是像mysqldump导出一个sql单一文件



2.查看mydumper线程数.这里发现默认有4个线程

```
[root@mysql ~]# ps aux | grep mydumper | grep -v grep
root     29941 20.5  0.0 337684 16488 pts/4    Sl+  22:35   1:36 ./mydumper -u root -p -B test -o /data/test

[root@mysql ~]# ps -T 29941
  PID  SPID TTY      STAT   TIME COMMAND
29941 29941 pts/4    Sl+    0:00 ./mydumper -u root -p -B test -o /data/test
29941 29942 pts/4    Sl+    0:23 ./mydumper -u root -p -B test -o /data/test
29941 29943 pts/4    Sl+    0:22 ./mydumper -u root -p -B test -o /data/test
29941 29944 pts/4    Sl+    0:19 ./mydumper -u root -p -B test -o /data/test
29941 29945 pts/4    Sl+    0:32 ./mydumper -u root -p -B test -o /data/test
```

3.还原数据库

```
root@mysql mydumper-0.9.1]# ./myloader -u root -p 123456  -B test -d /data/test/
```

4.myloader也用了4个线程 

```
[root@mysql iqg_staging]# ps -T -p 397
  PID  SPID TTY          TIME CMD
  397   397 pts/4    00:00:00 myloader
  397   398 pts/4    00:00:00 myloader
  397   399 pts/4    00:00:01 myloader
  397   400 pts/4    00:00:01 myloader
  397   401 pts/4    00:00:00 myloader
```

---



### 三.xtrabackup

#### 概述:

Xtrabackup是一个对InnoDB做数据备份的工具，支持在线热备份（备份时不影响数据读写）,全量备份,增量备份、差量备份 .xtrabackup是物理层面的备份,对于大量数据(超过50G)的数据库来说比mysqldump的备份速度要快好几倍.

以下官方文档可以查看详细的xtrabackup介绍

介绍xtrabackup的工作过程: [xtrabackup介绍](https://www.percona.com/doc/percona-xtrabackup/LATEST/how_xtrabackup_works.html )

介绍xtrabackup的安装:[xtrabackup安装](https://www.percona.com/doc/percona-xtrabackup/LATEST/installation.html#installing-from-binaries)

---

#### 备份先决条件

创建一个用于备份数据库的账户,赋予基本权限 

```
mysql> CREATE USER 'bkpuser'@'localhost' IDENTIFIED BY 's3cret';
mysql> GRANT RELOAD, LOCK TABLES, PROCESS, REPLICATION CLIENT ON *.* TO
'bkpuser'@'localhost';mysql> FLUSH PRIVILEGES;
```



**全量备份**

命令很简单. --backup参数表示备份,--target-dir参数表示备份到某个路径下.

```
xtrabackup --user=bkpuser --password=123456 --user=bkpuser --password=123456 --backup --target-dir=/data/backups/
```

> --database参数表示指定备份的数据库,如果没有指定默认备份所有库



**全量备份的恢复**

step1: 恢复一个备份数据,需要先对备份的数据进行"prepare"工作.

命令如下,--prepare参数表示准备恢复一个备份数据

```
xtrabackup --user=bkpuser --password=123456 --prepare --target-dir=/data/backups/
```

step2:恢复已经"prepare"的数据.命令如下

```
xtrabackup --user=bkpuser --password=123456 --copy-back --target-dir=/data/backups/
```

> note.恢复一个数据库的注意事项:
>
> 1.如果不需要保留备份的数据.可以使用--move-back参数替换--copy-back..这将在恢复工作完成后,删除备份数据
>
> 2.在备份之间必须要先关闭mysql数据库,并且删除(或者重命名)mysql的数据目录.建立一个空的数据文件目录.
>
> 3.恢复完成后,需要修改数据库文件目录的权限为mysql.mysql.否则无法启动数据库

---



**增量备份**

xtrabackup的第一次增量备份需要基于上一次的全量备份基础上进行增量备份.

命令和全量备份没有太大区别,--incremental-basedir参数表示这是一次增量备份.

```
xtrabackup --user=bkpuser --password=123456 --backup --target-dir=/data/backups/inc1 \
--incremental-basedir=/data/backups/base
```

> --incremental-basedir(/data/backups/base)是上一次全备目录.
>
> --target-dir(/data/backups/inc1)是本次增量备份目录 



第二次,第三次....等等后续的增量需要基于上一次的增量备份基础上进行.例如第二个增量备份命令:

```
xtrabackup --user=bkpuser --password=123456 --backup --target-dir=/data/backups/inc2 \
--incremental-basedir=/data/backups/inc1
```

第n次增量备份的basedir为n-1次增量备份的文件路径

---

**增量恢复**

增量备份的prepare和全备完全不一样.增量的Prepare步骤如下:

step1: prepare一个全备.需要携带--apply-log-only参数.命令如下:

```
$ xtrabackup --user=bkpuser --password=123456 --prepare --apply-log-only --target-dir=/data/backups/base
```

step2:prepare第一个增量备份..所有增量备份的prepare都是基于全备基础上

```
$ xtrabackup --user=bkpuser --password=123456 --prepare --apply-log-only --target-dir=/data/backups/base \
--incremental-dir=/data/backups/inc1
```

对于第N个增量备份:

```
$ xtrabackup --user=bkpuser --password=123456 --prepare --apply-log-only --target-dir=/data/backups/base \
--incremental-dir=/data/backups/incN
```

但是对于最后一个增量备份的prepare.,注意不能使用--apply-log-only选项:

```
$ xtrabackup --user=bkpuser --password=123456 --prepare --target-dir=/data/backups/base \
--incremental-dir=/data/backups/inc(最后一个)
```

> 对于增量备份的prepare需要注意以下几点:
>
> 1.需要指定--apply-log-only参数
>
> 2.所有增量备份prepare的incremental-dir都是全备数据文件所在路径
>
> 3.最后一个增量备份的prepare不能使用apply-log-only参数



step3:一旦所有的增量备份都全部prepare以后,就可以像全备恢复一样恢复数据库:

```
xtrabackup --user=bkpuser --password=123456 --copy-back --target-dir=/data/backups/
```

---



**xtrabackup的压缩备份**

xtrabackup支持备份的压缩功能,而且可以指定压缩线程数.

语法如下:

```
--compress  #表示开启压缩功能
--compress-threads=x   #在compress的基础上结合此参数可以调整压缩的线程数
```

例如:

```
xtrabackup ---user=bkpuser --password=123456 --backup  --compress --compress-threads=4  --target-dir=/data/backups/
```

> note: 压缩的线程数只会影响压缩备份数据的时间长短,,而对备份后的数据容量没有任何影响



在实践工作中中,对于一个700M左右的mysql数据库.没有启用压缩功能和启用了压缩功能的备份文件容量大小如下:

```
[root@localhost tmp]# du -sh /var/lib/mysql
723M    /var/lib/mysql

没有启用压缩:
[root@localhost tmp]# du -sh backup 
615M    backup

启用了压缩功能:
[root@localhost tmp]# du -sh backups
101M    backups
```



**压缩备份的prepare工作**

在prepare之前需要先对压缩文件进行解压.需要使用 --decompress选项 

```
xtrabackup --decompress  --target-dir=/data/backups/
```

> note:
>
> 1.需要先确保服务器已经安装了qpress软件.此软件在percona的软件源中.可以直接使用Yum下载安装 
>
> 2.可以结合使用 --parallel参数并发解压缩多个数据库备份文件.
>
> 3.xtrabackup解压缩备份文件时,不会自动删除压缩文件.如果需要清除备份目录.需要使用--remove-original参数 



解压缩备份文件后,接下来的prepare和还原工作和全量/增量一致.

---

**xtrabackup备份脚本**

以下脚本是我们公司生产中使用的脚本.屏蔽了一些关键信息.

```
#!/bin/bash
##############################################################
# File Name: backup.sh
# Version: V1.0
# Author: huangyong
# Created Time : 2018-3-1 18:42:00
# Description: 数据库全量,增量备份脚本

#备份策略:每周一进行全备,其他日期备份当周的增量备份.每次全备前删除2周前的备份
#可扩展功能:打包备份文件.备份文件传输到远程服务器

#date:2018-04-15
#update:由于数据库/data磁盘已快满.所以备份只保留一周.

#date:2018-04-23
#update:增加如果备份失败则发邮件通知功能
#       增加自动删除备份日志功能

#data:2018-04-24
#update:1.增加xtrabackup自带的的备份压缩功能,且压缩线程数4.
#       2.全备完成后,打包整个全备的备份文件(暂时先不打包)
#       3.全备完成后,同步备份文件到BETA服务器
#       4.保留2份备份文件,也就是保留2周
#       5.将脚本的执行用户从root改到xxxx.

#date:2018-05-03
#update:修改N_变量的抓取inc增量备份目录的命令.之前用的是sort命令,经常会抓取到错误的inc增量备份目录
#        脚本执行用户改成root,因为xxxx用户没有权限访问mysql的数据文件目录
##############################################################

#获取脚本所存放目录
cd `dirname $0`
bash_path=`pwd`
#脚本名
me=$(basename $0)

#设置要备份的innodb数据库，用空格格开，空为备份所有库
databases=''

#定义变量
DATE=$(date +%W) #全年的第几周,一个星期为一个备份周期.备份根目录，其子目录：base为全量，inc1、inc2...为增量
TWO_WEEKS_AGO=$(echo ${DATE}-2|bc) #前两周前的备份
FULL_DATE=$(date +%F) #存储日志日期
DAY_DATE=$(date +%w) #判断一周的第几天
#MYSQL="mysql"  # mysql命令绝对路径或在PATH中
MYSQL_DATA_DIR="/data/mysql/data"  # 数据库目录
BACKUP_USER="bkpuser"  # 备份用户
PASSWD=$(cat /data/xtrabackup/password)  # 备份密码保存文件
BACK_FILE_DIR="/data/backups/${DATE}"  # 备份频率目录，此目录变化频率为备份一周期
LOG_P_DIR="/data/backup_logs" #备份日志根目录
LOG_DIR="/data/backup_logs/${FULL_DATE}"  # 备份过程日志目录
#LOG_ERR="${LOG_DIR}/mysql_backup_fail.log" #备份错误日志文件
LOG_FILE="${LOG_DIR}/mysql_backup.log"  #备份过程日志文件
email_user="xxxx"
ssh_server="xxxx"  # 远程备份服务器IP
ssh_server_dir="/data/tongjidb-mysqlbackup"  # 远程备份服务器目录
ssh_port="22"  # ssh端口
ssh_parameters="-o StrictHostKeyChecking=no -o ConnectTimeout=60"
ssh_user="xxxx"
ssh_command="ssh ${ssh_parameters} -p ${ssh_port}"
#scp_command="scp ${ssh_parameters} -P ${ssh_port}"

#定义保存日志函数
function save_log () {
        
	echo -e "#################[`date +%F\ %T`]	$* ####################" >> $LOG_FILE

}

#定义发送邮件函数
function send_mail () {
        echo $1 | mail -s $1  $email_user

}

#创建目录
[ ! -d "${BACK_FILE_DIR}" ] && mkdir -p ${BACK_FILE_DIR}
[ ! -d "${LOG_DIR}" ] && mkdir -p ${LOG_DIR}

function full_backup () {
	# 全量备份函数
	[ ! -z "$databases" ] && option="--databases=${databases}" || option="" 

	##############################MYSQL全库备份#########################
	/usr/bin/xtrabackup  --user=$BACKUP_USER --password=$PASSWD --compress --compress-threads=4 --backup --target-dir=${BACK_FILE_DIR}/base --datadir=${MYSQL_DATA_DIR} $option > $LOG_FILE 2>&1
	
        if [ $? -eq 0 ];then
             save_log "mysql full_backup succeed"
             chown -R mysql:mysql ${BACK_FILE_DIR}/base
        
        else
             save_log "mysql full_backup failed"
             #send_mail "mysql full_backup failed"
             exit 1   
        
        fi                
	###################################################################

          
}

function incremental_backup () {
    [ ! -z "$databases" ] && option="--databases=${databases}" || option=""

    cd  $BACK_FILE_DIR
    # 判断是否存在第一次增量备份目录inc1
    # 存在则获取最后一次增量备份目录incN，然后基于最后一次增量备份，做增量备份
    # 不存在则基于全量备份目录base做增量备份
    if [ -d "inc1" ];then
        N_=$(ls -l | awk -F 'inc' '/^d+.+inc[0-9]+$/{a[NR]=$NF;len=asort(a,sa)}END{print sa[len]}')
        N=$(echo $N_+1|bc)
        #增量备份 
        /usr/bin/xtrabackup --user=$BACKUP_USER --password=$PASSWD --backup --compress --target-dir=$BACK_FILE_DIR/inc$N \
        --incremental-basedir=$BACK_FILE_DIR/inc$N_ --datadir=$MYSQL_DATA_DIR $option > $LOG_FILE 2>&1
    else
        N="1"
        #增量备份 
        [ ! -d $BACK_FILE_DIR/base ] && save_log "incremental backup failed,no full_backup" && exit 1
        /usr/bin/xtrabackup --user=$BACKUP_USER --password=$PASSWD --backup --compress --target-dir=$BACK_FILE_DIR/inc$N \
        --incremental-basedir=$BACK_FILE_DIR/base --datadir=$MYSQL_DATA_DIR $option > $LOG_FILE 2>&1
    fi


    if [ $? -eq 0 ];then
             save_log "mysql inc${N}-backup successed"
             chown -R mysql:mysql ${BACK_FILE_DIR}/inc$N
        
        else
             save_log "mysql inc${N}-backup failed" 
            #send_mail "mysql inc${N}-backup failed"
             exit 1   
                        
    fi

    return 0
}

function rsync_backup_files () {
	#传输到远程服务器备份, 需要配置免密ssh认证
        
        #使用rsync将本地的/data/backups目录同步到BETA服务器.同时删除BETA服务器上2周前的备份目录
	rsync -az --delete /data/backups -e "${ssh_command}" $ssh_user@${ssh_server}:$ssh_server_dir
	[ $? -eq 0 ] && save_log "full-backuped rsync successed" || \
	{ save_log "backup rsync failed" ; send_mail "mysql backup rsync failed" ; }
}


#每周1进行全备.其他日期对本周一的全备做增量备份
if [ $DAY_DATE -eq 1 ];then
    
   
   #删除2周前的备份文件
   if [ 1 -le $TWO_WEEKS_AGO -a $TWO_WEEKS_AGO -lt 10 ];then #如果本周和2周前的数相减小于10,并且大于等于1,则相差的结果前加个0.比如07
        FILE_NAME=$(dirname $BACK_FILE_DIR)/0$TWO_WEEKS_AGO
        [ -d $FILE_NAME ] && rm -rf $FILE_NAME

   elif [ $TWO_WEEKS_AGO -ge 10 ];then  #如果两数相减等于两位数,直接删除文件
           FILE_NAME=$(dirname $BACK_FILE_DIR)/$TWO_WEEKS_AGO
           [ -d $FILE_NAME ] && rm -rf $FILE_NAME

   fi

   full_backup #调用全备
  
else
     incremental_backup #调用增备

fi

#删除7天前日志文件
find $LOG_P_DIR -type d -mtime +7 -exec rm -rf {} \;

rsync_backup_files
```







