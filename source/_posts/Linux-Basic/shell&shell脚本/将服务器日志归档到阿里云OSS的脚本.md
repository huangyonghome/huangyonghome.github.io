---
title: 将服务器日志归档到阿里云OSS的脚本  
date: 2020-06-30 00:44:48  
tags: scripts
categories: [Linux-Basic,shell&shell脚本 ]  
comments: true  
copyright: true  
---



## 将服务器日志归档到阿里云OSS的脚本


```
#!/bin/bash
#Descripion: upload trade center logs file to Aliyun OSS
#Author: HuangYong
#date: 2019-04-29 

buket=mg-tradecenter-log-archived #OSS Buket name
year=$(date +%Y) #年份
year_dir=log-archived-${year} #OSS年份目录
month=$(date +%m) #月份
ossutil64_dir=/home/work

yesterday_logtime=$(date +%Y%m%d --date="-1 day") #upload yeasterday logfile
log_dir=/data/logs/apps/trade-center/trade-center # tradecenter log file parent dir
log_prefix="trade-center.log" #logfile prefix
hostname=api1 #tradecenter server

#判断是否安装ossutil64工具
[ ! -f "/home/work/ossutil64" ] && echo "请安装ossutil64软件" && exit 1

#判断年份目录是否存在,不存在则创建
if ! `${ossutil64_dir}/ossutil64 ls oss://${buket}/${year_dir}/ > /dev/null 2>&1`;then
   ${ossutil64_dir}/ossutil64 mkdir oss://${buket}/${year_dir}/
fi

#判断月份目录是否存在,不存在则创建
if ! `${ossutil64_dir}/ossutil64 ls oss://${buket}/${year_dir}/${month} > /dev/null 2>&1`;then
   ${ossutil64_dir}/ossutil64 mkdir oss://${buket}/${year_dir}/${month}/
fi


# 打包昨天的日志文件

cd $log_dir

for log_type in "debug" "error" "netrcd-admin" "netrcd-callback" "netrcd-gateway" "netrcd-notify" "script";do

    log_name="${log_prefix}.${log_type}.$yesterday_logtime"
    if [ -f $log_name ];then
        tar -zc -f ${hostname}.${log_name}.tar.gz  $log_name
    fi
done

# 上传文件到OSS

${ossutil64_dir}/ossutil64 cp $log_dir --include="${hostname}*.tar.gz" -r -f oss://${buket}/${year_dir}/${month}/

#上传完成后,删除打包的日志
[ $? == 0 ] && rm -f $log_dir/${hostname}* 
```