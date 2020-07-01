---
title: mysql数据库使用xtrabackup全量,增量备份  
date: 2020-06-30 00:44:48  
tags: scripts
categories: [Linux-Basic,shell&shell脚本 ]  
comments: true  
copyright: true  
---

## mysql数据库使用xtrabackup全量,增量备份

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
#       5.将脚本的执行用户从root改到work.

#date:2018-05-03
#update:修改N_变量的抓取inc增量备份目录的命令.之前用的是sort命令,经常会抓取到错误的inc增量备份目录
#        脚本执行用户改成root,因为work用户没有权限访问mysql的数据文件目录
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
BACKUP_USER="tongji"  # 备份用户
PASSWD=$(cat /data/xtrabackup/password)  # 备份密码保存文件
BACK_FILE_DIR="/data/backups/${DATE}"  # 备份频率目录，此目录变化频率为备份一周期
LOG_P_DIR="/data/backup_logs" #备份日志根目录
LOG_DIR="/data/backup_logs/${FULL_DATE}"  # 备份过程日志目录
#LOG_ERR="${LOG_DIR}/mysql_backup_fail.log" #备份错误日志文件
LOG_FILE="${LOG_DIR}/mysql_backup.log"  #备份过程日志文件
email_user="huangyong@doweidu.com"
ssh_server="10.25.2.85"  # 远程备份服务器IP
ssh_server_dir="/data/tongjidb-mysqlbackup"  # 远程备份服务器目录
ssh_port="5822"  # ssh端口
ssh_parameters="-o StrictHostKeyChecking=no -o ConnectTimeout=60"
ssh_user="work"
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

