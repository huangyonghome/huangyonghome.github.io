---
title: Ansible+zabbix利用自动发现监控Linux服务器磁盘I/O
date: 2018-07-13 08:59:58
tags:  zabbix
categories: Linux-Service
comments: true
copyright: true
---

### Ansible+zabbix利用自动发现监控Linux服务器磁盘I/O

#### 背景

公司集群服务器一共有6台.

系统版本: Linux CentOS 7.4 

zabbix版本: zabbix 3.4

<!--more-->

---

#### 监控方案

利用iostat命令监控磁盘的IO情况.下面简单介绍一下iostat命令的常用参数

```
-c           #仅显示CPU统计信息.与-d选项互斥.
-d           #仅显示磁盘统计信息.与-c选项互斥.
-k           #以K为单位显示每秒的磁盘请求数,默认单位块.
-t            #在输出数据时,打印搜集数据的时间.
-V           #打印版本号和帮助信息.
-x           #输出扩展信息.
```



iostat显示结果的详细属性值说明:

```
rrqm/s： 每秒进行 merge 的读操作数目.即 delta(rmerge)/s

wrqm/s： 每秒进行 merge 的写操作数目.即 delta(wmerge)/s

r/s： 每秒完成的读 I/O 设备次数.即 delta(rio)/s

w/s： 每秒完成的写 I/O 设备次数.即 delta(wio)/s

rsec/s： 每秒读扇区数.即 delta(rsect)/s

wsec/s： 每秒写扇区数.即 delta(wsect)/s

rkB/s： 每秒读K字节数.是 rsect/s 的一半,因为每扇区大小为512字节.(需要计算)

wkB/s： 每秒写K字节数.是 wsect/s 的一半.(需要计算)

avgrq-sz：平均每次设备I/O操作的数据大小 (扇区).delta(rsect+wsect)/delta(rio+wio)

avgqu-sz：平均I/O队列长度.即 delta(aveq)/s/1000 (因为aveq的单位为毫秒).

await： 平均每次设备I/O操作的等待时间 (毫秒).即 delta(ruse+wuse)/delta(rio+wio)

svctm： 平均每次设备I/O操作的服务时间 (毫秒).即 delta(use)/delta(rio+wio)

%util： 一秒中有百分之多少的时间用于 I/O 操作,或者说一秒中有多少时间 I/O 
```

---

#### zabbix监控的shell脚本

1. **磁盘发现脚本**


```
vim disk_discovery.sh

#!/bin/bash
diskarray=(`cat /proc/diskstats |grep -E "\bsd[a-z]\b|\bxvd[a-z]\b|\bvd[a-z]\b"|awk '{print $3}'|sort|uniq  2>/dev/null`)
length=${#diskarray[@]}
printf "{\n"
printf  '\t'"\"data\":["
for ((i=0;i<$length;i++))
do
        printf '\n\t\t{'
        printf "\"{#DISK_NAME}\":\"${diskarray[$i]}\"}"
        if [ $i -lt $[$length-1] ];then
                printf ','
        fi
done
printf  "\n\t]\n"
printf "}\n"
```



2.监控磁盘IO脚本

```
vim disk_status.sh

#/bin/sh
Device=$1
DISK=$2
case $DISK in
         rrqm)
            iostat -dxkt 1 2|grep "\b$Device\b"|tail -1|awk '{print $2}'
            ;;
         wrqm)
            iostat -dxkt 1 2|grep "\b$Device\b"|tail -1|awk '{print $3}'
            ;;
          rps)
            iostat -dxkt 1 2|grep "\b$Device\b"|tail -1|awk '{print $4}'
            ;;
          wps)
            iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{print $5}'
            ;;
        rKBps)
            iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{print $6}'
            ;;
        wKBps)
            iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{print $7}'
            ;;
        avgrq-sz)
            iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{print $8}'
            ;;
        avgqu-sz)
            iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{print $9}'
            ;;
        await)
            iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{print $10}'
            ;;
        svctm)
            iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{print $13}'
            ;;
         util)
            iostat -dxkt |grep "\b$Device\b" |tail -1|awk '{print $14}'
            ;;
esac
```

>我的生产环境中,由于读写IO较大,.所以rKBps和wKBps的读写量非常多大,修改上面的rkBps和wKBp.

```
 rKBps) iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{ r=$6/1024} END {print r}'
 wKBps) iostat -dxkt 1 2|grep "\b$Device\b" |tail -1|awk '{ w=$7/1024} END {print w}'
```

> 这样修改后,后续的Zabbix的相关监控项单位也要改成MB/s.因为我们已将将原来的kBps单位换算成了MBps



**3.zabbix客户端配置文件**

```
vim userparameter_disk.conf

UserParameter=disk.discovery[*],/etc/zabbix/script/disk_discovery.sh
UserParameter=disk.status[*],/etc/zabbix/script/disk_status.sh $1 $2
```

---

#### 利用Aansible将脚本下发到远程服务器

将以上3个脚本文件,放到ansible的files目录下.

ansible的playbook编排如下:

```
---
 - hosts: bi
   remote_user: root

   tasks:
     - name: "create dir of script files"
       file: path=/etc/zabbix/script state=directory

     - name: copy  script on remote servers
       copy: src=files/{{ item }} dest=/etc/zabbix/script/{{ item }}  mode=0755 owner=zabbix group=zabbix
       with_items:
           - disk_discovery.sh
           - disk_status.sh

     - name: copy userparameter config on remote servers
       copy: src=files/userparameter_disk.conf dest=/etc/zabbix/zabbix_agentd.d/userparameter_disk.conf mode=0644

     - name: restart zabbix agent
       command: systemctl restart zabbix-agent
```

> 可能需要检查一下zabbix的客户端配置文件是否有下面这个配置,一般默认安装后都会有下面这个配置

```
[root@server-6 zabbix]$grep "Include" /etc/zabbix/zabbix_agentd.conf | grep -v "\#"
Include=/etc/zabbix/zabbix_agentd.d/*.conf
```

---

#### zabbix web控制台配置

参考这篇文档:[zabbix监控磁盘IO](http://blog.51cto.com/jiay1/2064696)



