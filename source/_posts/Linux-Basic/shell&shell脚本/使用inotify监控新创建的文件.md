---
title: 使用inotify监控新创建的文件
date: 2018-06-12 22:59:58
tags:  scripts
categories: [Linux-Basic,shell&shell脚本 ]
comments: true
copyright: true
---

## 使用inotify监控新创建的文件



```

#!/bin/bash
#description:监测某个路径下的是否有新创建的文件.如果有新创建的文件,则判断是否以某个固定的前缀开头.
#            如果是需要监测的文件,则发邮件通知给用户,文件已经创建,且通告文件名,文件创建日期等信息.
#            此脚本只监测一台服务器,一个目标文件夹,一个create的属性...........
#date:2018-03-29

#v2.0 多了个数组,有多个文件前缀.只要匹配其中任何一个就是监测的文件目标

src=/tmp
file_start_with=("SBD_sbdBalFinancing_LDAYSTMTDTL","INCOME_INFO_SBD_227","PRODUCT_INFO_SBD_227","TRANS_INFO_SBD")
email_user="mailhuangyong@163.com"
log_file="/tmp/monitorfile.log"

if ! `which inotifywait > /dev/null 2>&1` ;then
   echo "sorry,inotify is not installed "> $log_file
   exit 1
fi

if ! `which mail > /dev/null 2>&1`;then
   echo "sorry,mail is not installed"> $log_file
fi

#判断文件名的前缀是否符合监测目标数组函数
function check_file () {
    file_start=$(echo $1 |  sed -r 's/(.*)_[0-9]*\.zip/\1/')
    if [ ${file_start_with[@]/$file_start/} != ${file_start_with[@]} ];then
       return 0
    else
      return 1
    fi

}


#发送邮件
function send_mail () {
    echo -e "Filename is :             $1\nFile create time is :  $2" |mail -s "receive file succeed!" $email_user

}


#inotify监测目录下文件是否有改动.主要监测:文件名或者目录创建(这里我没有监测文件删除,修改,移动,权限等).
#将监测到的文件重定向到while循环.
inotifywait -rm --format '%Xe %f' -e create $src | while read file;do

   #获取文件名
   mon_file=$(echo $file | awk '{print $2}')
   #判断文件名是否符合固定前缀
   check_file $mon_file
   
   #接收check_file函数返回结果,如果是监测的文件已经被创建,就发送邮件
   if [ $? -eq 0  ];then
      file_time=$(stat $src/$mon_file  | awk -F '[" ".]' '/Change/ {print $2,$3}')
      send_mail $mon_file "$file_time"
      [[ $? -ne 0 ]] &&  echo "send email failed" > $log_file
   fi

done
```

