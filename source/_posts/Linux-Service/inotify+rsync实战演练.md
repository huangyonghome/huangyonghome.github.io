---
title:  inotify+rsync实战演练
date: 2018-06-24 12:59:58
tags:  Linux,rsync
categories: [Linux-Service ]
comments: true
copyright: true
---



## inotify+rsync实战演练

#### 试验目的:

演练rsync结合inotify实现服务端目录内文件有变动(包括修改,删除,创建)时,自动立即同步到客户端 

#### 试验环境: 

centos6.5 192.168.10.89 -----角色:文件同步服务器.原始文件服务器.rsync客户端,inotify服务器  
centos 6.5 192.168.10.103-----角色:文件同步客户端,由文件服务器自动向客户端同步 

关于rsync和inotify介绍和具体用法.请参考其他笔记内容

<!--more-->

---

#### 实战步骤

一.在inotify服务器安装inotify-tools工具

[下载链接](https://sourceforge.net/projects/inotify-tools/?source=typ_redirect)

安装过程简单:

```
tar zxvf inotify-tools-3.13.tar.gz
cd inotify-tools-3.13
./configure --prefix=/usr/local/inotify
make && make install 

vim /etc/profile
在结尾处加上:
export PATH=$PATH:/usr/local/inotify/bin

应用profile文件:
source /etc/profile
```



二.演示inotify使用方法: 

执行命令:

```
inotifywait -rm --format '%Xe %w%f' -e modify,create,delete,attrib /tmp/data
```

命令输出:

```
[root@oracle inotify-tools-3.13]# inotifywait -rm --format '%Xe %w%f' -e modify,create,delete,attrib /tmp/data
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.

```

解析:

**inotifywait** : 持续监控文件的状态变化

**-r** :  递归监控目录下的所有文件,包括子目录.

> Note:如果要监控的目录中文件数量巨大，则通常需要修改/proc/sys/fs/inotify/max_users_watchs内核参数，因为其默认值为8192.

**-m**: 实现持续监控

**--format** 显示格式.                

- %X----事件以"X"分隔.                
- %e----显示事件(比如CREATE,MODIFY等),               
- %w----显示文件名               
- %f-----显示目录
-  -e: 表示检测哪些事件       

**/tmp/data**-------监测的目录路径 

再开启一个终端,然后在/tmp/data目录下新建一些文件:

```
[root@localhost data]# touch {x,y,z,u,v,w}.txt
```

 inotify输出如下:

检测到了文件变化.第一列是事件类型.有CREATE,ATTRIB. 第二列是文件的完整路径

```
[root@oracle inotify-tools-3.13]# inotifywait -rm --format '%Xe %w%f' -e modify,create,delete,attrib /tmp/data
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
CREATE /tmp/data/x.txt
ATTRIB /tmp/data/x.txt
CREATE /tmp/data/y.txt
ATTRIB /tmp/data/y.txt
CREATE /tmp/data/z.txt
ATTRIB /tmp/data/z.txt
CREATE /tmp/data/u.txt
ATTRIB /tmp/data/u.txt
CREATE /tmp/data/v.txt
ATTRIB /tmp/data/v.txt
CREATE /tmp/data/w.txt
ATTRIB /tmp/data/w.txt

```



再试着删除所有文件:

```
[root@localhost data]# rm -rf  {x,y,z,u,v,w}.txt
```

 inotify检测到DELETE事件:

```
DELETE /tmp/data/x.txt
DELETE /tmp/data/y.txt
DELETE /tmp/data/z.txt
DELETE /tmp/data/u.txt
DELETE /tmp/data/v.txt
DELETE /tmp/data/w.txt
```

试试创建和删除目录检测到CREATE和DELETE的目录事件:

```
CREATEXISDIR /tmp/data/test

```

试试修改文件内容

```
[root@localhost data]# echo "haha" > 1.txt
```

检测到MODIFY事件:

```
CREATE /tmp/data/1.txt
MODIFY /tmp/data/1.txt
```

---



基本用法就介绍到这里.下面实战演练inotify+rsync结合做目录文件同步 

在inotify编写脚本文件:

```
以下是工作在相对路径下

[root@localhost ~]# vim inotify_rsync.sh
#!/bin/bash
src=/tmp/data
des="/" #由于工作在相对路径下,会同步目录名.所以目的路径为/根
ip=192.168.10.103
user=root
cd $src #切换进工作目录
#inotify监测目录下文件是否有改动.主要监测:文件名或者目录修改,创建,删除,移动,文件内容修改.(这里我没有监测文件权限属性发送变化).
#将监测到的文件重定向到while循环.
inotifywait -mr --format '%Xe %w%f' -e modify,create,delete,close_write,move $src | while read file;do
   #获取Inotify的监测事件.有CREATE,MODIFY,DELETE等
   ino_event=$(echo $file | awk '{print $1}')
   #获取inotify监测到的变化文件
   ino_file=$(echo $file | awk '{print $2}')
   echo $file
   #if的正则匹配.如果ino_event变量的内容匹配CREATE开头.那么就条件为true.其实就等于 $ino_event == "CREATE*"
   if [[ $ino_event =~ "CREATE" ]] || [[ $ino_event =~ "MODIFY" ]] || [[ $ino_event =~ "CLOSE_WRITE" ]] || [[ $ino_event =~ "MOVED_TO" ]];then
         echo "CREATE or MODIFY or CLOSE_WRITE or MOVED_TO"
         #如果是文件有变化,则利用rsync同步该文件的父目录到远程主机相关目录下.这里使用了ssh协议.需要提前复制本机公钥到目的主机
         echo $(dirname $ino_file)
         /usr/bin/rsync -avzR -e ssh $(dirname $ino_file) $user@$ip:$des
   elif [[ $ino_event =~ "DELETE" ]] || [[ $ino_event =~ "MOVED_FROM" ]];then
         echo "Delete or Moved_From"
        #如果是文件删除,或者移动到其他地方.则利用rsync删除远程主机上的该文件
        /usr/bin/rsync -avzR --delete $(dirname $ino_file) $user@$ip:$des
   fi
done
```



```
以下是工作在绝对路径下:

[root@localhost ~]# vim inotify_rsync.sh
#!/bin/bash
src=/tmp/data
des=/tmp  #由于会同步/tmp/data目录.所以目的路径只需要指定/tmp目录
ip=192.168.10.103
user=root
#inotify监测目录下文件是否有改动.主要监测:文件名或者目录修改,创建,删除,移动,文件内容修改.(这里我没有监测文件权限属性发送变化).
#将监测到的文件重定向到while循环.
inotifywait -mr --format '%Xe %w%f' -e modify,create,delete,close_write,move $src | while read file;do
   #获取Inotify的监测事件.有CREATE,MODIFY,DELETE等
   ino_event=$(echo $file | awk '{print $1}')
   #获取inotify监测到的变化文件
   ino_file=$(echo $file | awk '{print $2}')
   echo $file
   #if的正则匹配.如果ino_event变量的内容匹配CREATE开头.那么就条件为true.其实就等于 $ino_event == "CREATE*"
   if [[ $ino_event =~ "CREATE" ]] || [[ $ino_event =~ "MODIFY" ]] || [[ $ino_event =~ "CLOSE_WRITE" ]] || [[ $ino_event =~ "MOVED_TO" ]];then
         echo "CREATE or MODIFY or CLOSE_WRITE or MOVED_TO"
         #如果是文件有变化,则利用rsync同步该文件的父目录到远程主机相关目录下.这里使用了ssh协议.需要提前复制本机公钥到目的主机
         /usr/bin/rsync -avz -e ssh $ino_file $user@$ip:$des
   elif [[ $ino_event =~ "DELETE" ]] || [[ $ino_event =~ "MOVED_FROM" ]];then
         echo "Delete or Moved_From"
        #如果是文件删除,或者移动到其他地方.则利用rsync删除远程主机上的该文件
        /usr/bin/rsync -avz --delete $ino_file $user@$ip:$des
   fi
done
```

> Note:此脚本中的rsync使用的是ssh协议传输.而不是守护模式.所以需要实现传输本地的公钥到远程主机相关用户下 



运行脚本:

```
[root@localhost ~]# ./inotify_rsync.sh
```

在/tmp/data目录内创建文件:

```
[root@localhost data]# touch {1,2,3,4,5,6}.txt
```



脚本输出:

```
[root@oracle ~]# ./inotify_rsync.sh
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
CLOSE_WRITEXCLOSE /tmp/data/1.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list
/tmp/
/tmp/data/
/tmp/data/1.txt
/tmp/data/2.txt
/tmp/data/3.txt
/tmp/data/4.txt
/tmp/data/5.txt
/tmp/data/6.txt
/tmp/data/test/

sent 388 bytes  received 138 bytes  350.67 bytes/sec
total size is 5  speedup is 0.01
CREATE /tmp/data/2.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  338.00 bytes/sec
total size is 5  speedup is 0.03
CLOSE_WRITEXCLOSE /tmp/data/2.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  338.00 bytes/sec
total size is 5  speedup is 0.03
CREATE /tmp/data/3.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  112.67 bytes/sec
total size is 5  speedup is 0.03
CLOSE_WRITEXCLOSE /tmp/data/3.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  338.00 bytes/sec
total size is 5  speedup is 0.03
CREATE /tmp/data/4.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  112.67 bytes/sec
total size is 5  speedup is 0.03
CLOSE_WRITEXCLOSE /tmp/data/4.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  338.00 bytes/sec
total size is 5  speedup is 0.03
CREATE /tmp/data/5.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  338.00 bytes/sec
total size is 5  speedup is 0.03
CLOSE_WRITEXCLOSE /tmp/data/5.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  112.67 bytes/sec
total size is 5  speedup is 0.03
CREATE /tmp/data/6.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  338.00 bytes/sec
total size is 5  speedup is 0.03
CLOSE_WRITEXCLOSE /tmp/data/6.txt
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 154 bytes  received 15 bytes  338.00 bytes/sec
total size is 5  speedup is 0.03

```



 在172.16.1.120客户端的/tmp/data目录下查看文件: 文件已成功复制:

```
[root@www ~]$ll /tmp/data
total 8
-rw-r--r--. 1 root root    5 Jun 24 13:27 1.txt
-rw-r--r--. 1 root root    0 Jun 24 13:27 2.txt
-rw-r--r--. 1 root root    0 Jun 24 13:27 3.txt
-rw-r--r--. 1 root root    0 Jun 24 13:27 4.txt
-rw-r--r--. 1 root root    0 Jun 24 13:27 5.txt
-rw-r--r--. 1 root root    0 Jun 24 13:27 6.txt
```



演示在Inotify服务上删除刚创建的文件: 监测到文件删除

```
[root@oracle ~]# ./inotify_rsync.sh
Setting up watches.  Beware: since -r was given, this may take a while!
Watches established.
DELETE /tmp/data/1.txt
Delete or Moved_From
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list
/tmp/data/
deleting tmp/data/6.txt
deleting tmp/data/5.txt
deleting tmp/data/4.txt
deleting tmp/data/3.txt
deleting tmp/data/2.txt
deleting tmp/data/1.txt

sent 87 bytes  received 18 bytes  70.00 bytes/sec
total size is 0  speedup is 0.00
DELETE /tmp/data/2.txt
Delete or Moved_From
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 84 bytes  received 15 bytes  198.00 bytes/sec
total size is 0  speedup is 0.00
DELETE /tmp/data/3.txt
Delete or Moved_From
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 84 bytes  received 15 bytes  66.00 bytes/sec
total size is 0  speedup is 0.00
DELETE /tmp/data/4.txt
Delete or Moved_From
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 84 bytes  received 15 bytes  198.00 bytes/sec
total size is 0  speedup is 0.00
DELETE /tmp/data/5.txt
Delete or Moved_From
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 84 bytes  received 15 bytes  198.00 bytes/sec
total size is 0  speedup is 0.00
DELETE /tmp/data/6.txt
Delete or Moved_From
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list

sent 84 bytes  received 15 bytes  66.00 bytes/sec
total size is 0  speedup is 0.00

```



在172.16.1.120服务器上查看/tmp/data目录.下面没有任何文件

```
[root@www ~]$ll /tmp/data
total 4
drwxr-xr-x. 2 root root 4096 Jun 24 13:17 test
```



演示:新建一个目录.且在该目录下创建内容 脚本输出:

```
CREATEXISDIR /tmp/data/haha
CREATE or MODIFY or CLOSE_WRITE or MOVED_TO
/tmp/data
Nasty PTR record "172.16.1.120" is set up for 172.16.1.120, ignoring
sending incremental file list
/tmp/data/
/tmp/data/haha/

sent 100 bytes  received 22 bytes  244.00 bytes/sec
total size is 0  speedup is 0.00

```

目录已经被同步

```
[root@www ~]$ll /tmp/data
total 8
drwxr-xr-x. 2 root root 4096 Jun 24 13:33 haha
drwxr-xr-x. 2 root root 4096 Jun 24 13:17 test

```



