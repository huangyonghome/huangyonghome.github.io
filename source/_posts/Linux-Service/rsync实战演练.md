---
title:  rsync实战演练
date: 2018-06-24 12:59:58
tags:  rsync
categories: [Linux-Service ]
comments: true
copyright: true
---



## rsync实战演练

rsync是一个远程数据同步工具,可以快速同步多台主机的文件,且只同步有差异的部分.非常强大的工具 

#### 实战环境:

服务端:192.168.10.89    
客户端:192.168.10.103 

rsync不需要安装,默认就自带.

关于rsync的命令想法,可以参考其他笔记.

> note: 以下教程都是讲述客户端从远程服务器同步数据到本地.类似于下载行为. 如果需要将本地的文件同步到远程服务器.有点类似于上传行为.则需要改变命令.   



<!--more-->

下列命令表示了上传和下载的使用区别.

> note:在使用rsyn同步前必须要千万小心.因为如果命令搞反.可能会出现意外的严重后果.例如将对方的文件同步到本地数据目录



1.下列的命令将对方(192.168.10.89)的/var/www/abc目录同步到本地的/root/rsync目录下:

```
rsync -avz --progress -e ssh root@192.168.10.89:/var/www/abc /root/rsync
```



2.下列命令表示将本地的/tmp/backups目录同步到对方(192.168.10.89)的/var/www/abc目录下 

```
rsync -avz --progress /tmp/backups -e ssh root@192.168.10.89:/var/www/abc  
```

3.另外.如果远程服务器的ssh不是默认22端口.则需要改成: 

```
rsync -avz --progress /root/rsync -e "ssh -p 端口 " root@192.168.10.89:/var/www/abc
```

---

#### Rync的使用方法介绍:

 **一.以服务端的方式启动rsync进程** 

1.编辑/etc/rsyncd.conf配置文件-------这个文件默认没有,需要自己写入

```
vim /etc/rsyncd.conf
#全局参数.所有模块生效配置#
uid = nobody
gid = nobody
use chroot = no
max connections = 4
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log
#模块参数.一个模块代表一个路径#
[www]
path = /var/www/abc/   #路径目录,注意必须是一个/结尾的目录
ignore errors          #忽略错误信息
read only = yes       #服务端只读,客户端只能和服务端同步,不能上传文件
list = no          #是否允许客户端列出服务端此路径下的文件
hosts allow = 192.168.10.0/24  #允许哪个网络上的客户端同步
auth users = backup    #认证用户名
secrets file = /etc/rsync_server.pas #指定一个密码文件路径.此文件内容为username:password. 而且此文件必须和启动rsync的用户是同一个用户.且权限为600
```



2.编辑密码文件:

```
[root@localhost ~]# cat /etc/rsync_server.pas
backup:jesse
[root@localhost ~]# ll /etc/rsync_server.pas
-rw------- 1 root root 13 Dec 27 16:46 /etc/rsync_server.pas
```

> Note:此文件必须为600权限.且和rsync进程的用户相同.比如如果是root启动的rsync服务.则此文件属主也必须是root  



3.启动rsync服务,以daemon方式启动

```
rsync --daemon
```

 rsync --daemon默认监控在873端口

```
默认监控在873端口:
[root@localhost ~]# netstat -tulnp | grep rsync
tcp        0      0 0.0.0.0:873                 0.0.0.0:*                   LISTEN      2034/rsync
tcp        0      0 :::873                      :::*                        LISTEN      2034/rsync
```

4.在/var/www/abc目录下写入一个测试文件夹

```
[root@localhost ~]# cat /var/www/abc/rsync.test
haha
this is for test rsync
the nginx02 is rsync server.the nginx 01 (ip:192.168.10.103) is a client
I am going to see whether this file will be sync to the client or not!
```

5.客户端同步文件 先定义密码.改成600权限

```
[root@localhost ~]# cat /etc/rsync_server.pas
jesse
[root@localhost ~]# ll /etc/rsync_server.pas
-rw------- 1 root root 6 Dec 27 16:47 /etc/rsync_server.pas
```

> Note:我在这里踩到一个大坑.客户端的密码文件只需要包含密码.不能像服务端一样指定username:password.不然会提示验证失败 



6.执行rsync命令

```
/usr/bin/rsync -avz  --progress backup@192.168.10.89::www /var/www/abc/ --password-file=/etc/rsync_server.pas

-a ----保持文件权限
-v ----详细显示
-z -----启用压缩
--progress --显示备份过程
backup@ ----表示用backup用户认证
::www  --------注意这里有2个冒号.表示同步服务器上的www模块
/var/www/abc ---表示同步到本地这个目录下
--password-file ---表示用这个文件内的密码去认证
```

可是遇到和上面一样的坑:

```
[root@localhost ~]# /usr/bin/rsync -avz  --progress backup@192.168.10.89::www /var/www/abc/ --password-file=/etc/rsync_server.pas
@ERROR: auth failed on module www
```

![rsync](https://img1.jesse.top/static/images/Linux/rsync.png)

这个坑,至少坑了我5个小时.反复的确认selinux是否关闭,配置文件是否错误,密码文件和密码文件权限等最后才发现,原来配置文件的配置语句不能用注释  

注释内容只能单独一行存在.修改配置文件:

```
[root@localhost ~]# vim /etc/rsyncd.conf
#全局参数.所有模块生效配置#
uid = nobody
gid = nobody
use chroot = no
max connections = 4
pid file = /var/run/rsyncd.pid
lock file = /var/run/rsyncd.pid
log file = /var/log/rsyncd.log
#模块参数.一个模块代表一个路径#
[www]
path = /var/www/abc/
ignore errors
read only = yes
list = no
hosts allow = 192.168.10.0/24
auth users = backup
secrets file = /etc/rsync.secrets
```

客户端重新执行命令:

```
[root@localhost ~]# /usr/bin/rsync -avz  --progress backup@192.168.10.89::www /var/www/abc/ --password-file=/etc/rsync_server.pas
receiving incremental file list
./
index.html
          91 100%   88.87kB/s    0:00:00 (xfer#1, to-check=2/4)
rsync.test
         173 100%  168.95kB/s    0:00:00 (xfer#2, to-check=1/4)
test.jpg
      185883 100%   25.32MB/s    0:00:00 (xfer#3, to-check=0/4)
sent 115 bytes  received 186303 bytes  372836.00 bytes/sec
total size is 186147  speedup is 1.00
[root@localhost ~]#
可以看到同步了rsync.test文件过来了.另外其他2个文件也一并同步过来
```

---

#### 演示: 服务端文件名不修改.往文件内新增文件.观察rsync是否能实行增量同步



 1.在rsync服务端内的/var/www/abc/index.html文件新增一行内容:

```
[root@rabbitmqnode0 abc]# vim index.html
hello.This is nginx server for www.abc.com
hello world!
add a new line to test rsync   #新增一行
```



 2.客户端在原有同步后的基础上再次执行命令:可以看见.只同步了Index.html文件.其他文件并没有复制

```
[root@rabbitmqnode1 ~]# /usr/bin/rsync -avz  --progress backup@172.16.1.130::www /var/www/abc/ --password-file=/etc/rsync_server.pas
receiving incremental file list
./
index.html
          85 100%   83.01kB/s    0:00:00 (xfer#1, to-check=1/3)
sent 83 bytes  received 251 bytes  668.00 bytes/sec
total size is 106558  speedup is 319.04
```

查看文件:

```
[root@rabbitmqnode1 ~]# cat /var/www/abc/index.html
hello.This is nginx server for www.abc.com
hello world!
add a new line to test rsync

可见.即便文件名一致.只要文件内容有变化,仍然会同步到客户端.    
```

---



 #### 二.以ssh方式同步文件

 1.关掉服务端的rsync进程

```
ps -ef | grep rsync
root       2014      1  0 21:28 ?        00:00:00 rsync --daemon
root       6334   2588  0 22:37 pts/1    00:00:00 grep rsync
[root@localhost ~]# kill -9 2014
[root@localhost ~]# rm -rf /var/run/rsyncd.pid
```



2.客户端直接执行命令:

```
[root@localhost ~]# rsync -avz --progress -e ssh root@192.168.10.89:/var/www/abc /root
root@192.168.10.89's password:
receiving incremental file list
abc/
abc/index.html
          91 100%   88.87kB/s    0:00:00 (xfer#1, to-check=2/4)
abc/rsync.test
         173 100%  168.95kB/s    0:00:00 (xfer#2, to-check=1/4)
abc/test.jpg
      185883 100%   25.32MB/s    0:00:00 (xfer#3, to-check=0/4)
sent 72 bytes  received 186254 bytes  53236.00 bytes/sec
total size is 186147  speedup is 1.00
[root@localhost ~]#
```

> -e 表示 使用ssh协议root@ 表示服务器的本地用户(注意,这里是服务器本地真实用户)..这里的用法和普通的scp命令一致 



可以看到文件已经同步过来了.且权限保持一致

```
[root@localhost ~]# ll /root/abc
total 192
-rw-r--r-- 1 root root     91 Dec 26 19:21 index.html
-rw-r--r-- 1 root root    173 Dec 27 15:07 rsync.test
-rw-r--r-- 1 root root 185883 Dec 26 19:21 test.jpg
```

> 注意:ssh模式不能像rsync daemon模式那样指定一个密码文件.如果想不输入密码.只能复制本机的公钥到rsync服务端相关用户下.

> Note:
>
> 1.一般在生产中 在客户端同步的时候还需要加入个  --delete参数.表示如果本机相关目录下有某个文件.而这个文件在服务端上没有.那么就删除.这是为了保持和服务端完全同步 
>
> 2.一般需要写一个crontab定时任务,每5分钟同步一次
>
> */5 * * * * /usr/bin/rsync -avz  --progress backup@192.168.10.89::www /var/www/abc/ --password-file=/etc/rsync_server.pas > /dev/null 2>&1   