---
title: 分析统计Nginx日志的响应时间 
date: 2020-06-30 00:44:48  
tags: scripts
categories: [Linux-Basic,shell&shell脚本 ]  
comments: true  
copyright: true  
---



## 分析统计Nginx日志的响应时间

Nginx日志格式如下:


```
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" $http_x_forwarded_for "$request_time"';
```

实际日志如下:

```
[work@iqg-yyq2 ~]$ head  /data/logs/nginx/iqg_api_v5.access.log
100.97.74.45 - - [11/Mar/2019:03:46:01 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"
100.117.85.154 - - [11/Mar/2019:03:46:01 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"
100.97.74.22 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.002"
100.117.85.96 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"
100.117.85.133 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"
100.117.85.172 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.002"
100.97.74.0 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"
100.97.73.184 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"
100.117.85.85 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"
100.97.74.62 - - [11/Mar/2019:03:46:02 +0800] "HEAD / HTTP/1.0" 200 0 "-" "-" - "0.001"

```

最后一行为响应时间.但是是个字符串,还不能直接用awk来统计

下面这个脚本用来统计响应时间:


```

#!/bin/bash

[ $# -ne 2 ] && echo "Usage: ./loganalysis.sh two parameters: logfile path  and cost time" && exit 0

[ ! -f "$1" ] && echo "the file doesn't exsit,please check again" && exit 0
logfile=$(basename $1)

[ "$2" -lt 0 ] && echo " the second parameter is not a digit" && exit 0
cost_time=$2

cat $1 | awk '{print $NF}'  | awk -F "\"" '{print $2}'  >  time.txt
echo "split request_time over!!!"

paste  -d " " $1 time.txt > new.log
echo "build new logfile over!!!"

awk '($NF>'$cost_time'){print $0}' new.log > slow-${logfile}
echo "please see slowtime in slow-${logfile}"

rm -f time.txt
rm -f new.log

# analyze the access frequence of API

echo "#############the access frequence of API ##################"
awk '{++S[$4]} END { for (i in S) print "URL:"i "\t" "access times:"S[i]}' slow-${logfile}
```

下面是统计iqg_api_v5.access.log这个日志响应时间超过1秒的记录

```
[work@iqg-yyq2 ~]$ ./loganalysis.sh /data/logs/nginx/iqg_api_v5.access.log 1
split request_time over!!!
build new logfile over!!!
please see slowtime in slowtime.txt!!!
[work@iqg-yyq2 ~]$
```

查看最终结果:

```
work@iqg-yyq2 ~]$ head slowtime.txt
100.117.85.64 [11/Mar/2019:14:14:15 "HEAD / 1.223
100.97.74.41 [11/Mar/2019:14:14:16 "HEAD / 1.984
100.97.74.104 [11/Mar/2019:14:14:16 "HEAD / 1.880
100.97.74.5 [11/Mar/2019:14:14:16 "HEAD / 1.758
100.117.85.100 [11/Mar/2019:14:14:16 "HEAD / 1.757
100.97.73.213 [11/Mar/2019:14:14:16 "HEAD / 1.767
100.117.85.160 [11/Mar/2019:14:14:16 "HEAD / 1.662
100.97.74.118 [11/Mar/2019:14:14:16 "HEAD / 1.566
100.97.73.141 [11/Mar/2019:14:14:16 "HEAD / 1.215
100.117.56.238 [11/Mar/2019:14:14:16 "HEAD / 1.132
[work@iqg-yyq2 ~]$
```
