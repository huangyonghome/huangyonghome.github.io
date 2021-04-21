---
title: rabbitmq集群版本升级
date: 2018-11-21 17:59:58
tags: MQ
categories: [Linux-分布式&消息队列,mq ]
comments: true
copyright: true
---



##  rabbitmq集群版本升级

### 介绍

本文档介绍rabbitmq所有节点从当前的3.6.3版本升级到3.6.5版本.



### 升级步骤

1.停止rabbitmq进程

```
service rabbitmq-server stop
```



2.备份延迟队列插件.(如果你安装了其他自定义插件,也需要先备份出来)

```
cp /usr/lib/rabbitmq/lib/rabbitmq_server-3.6.3/plugins/rabbitmq_delayed_message_exchange-0.0.1.ez  ~/
```

> 可以使用 yum -ql 软件包 (CentOS) 或者 dpkg -L 软件包来查找插件安装路径



3.下载软件包.本文档采用RPM包或者deb包安装

```
#Centos系统
https://github.com/rabbitmq/rabbitmq-server/releases/download/rabbitmq_v3_6_5/rabbitmq-server-3.6.5-1.noarch.rpm

#Ubuntu系统
https://github.com/rabbitmq/rabbitmq-server/releases/download/rabbitmq_v3_6_5/rabbitmq-server_3.6.5-1_all.deb
```

<!--more-->

4.在root用户下升级软件包

```
# CentOS系统
rpm -Uvh rabbitmq-server-3.6.5-1.noarch.rpm

#Ubuntu系统
dpkg -i rabbitmq-server_3.6.5-1_all.deb
```



安装后会报错,是正常现象,因为新版本没有插件

```
root@hsq-mq-node2-temp:~# dpkg -i rabbitmq-server_3.6.5-1_all.deb
(Reading database ... 171471 files and directories currently installed.)
Preparing to unpack rabbitmq-server_3.6.5-1_all.deb ...
Unpacking rabbitmq-server (3.6.5-1) over (3.6.3-1) ...
dpkg: warning: unable to delete old directory '/usr/lib/rabbitmq/lib/rabbitmq_server-3.6.3/plugins': Directory not empty
dpkg: warning: unable to delete old directory '/usr/lib/rabbitmq/lib/rabbitmq_server-3.6.3': Directory not empty
Setting up rabbitmq-server (3.6.5-1) ...
Job for rabbitmq-server.service failed because the control process exited with error code. See "systemctl status rabbitmq-server.service" and "journalctl -xe" for details.
invoke-rc.d: initscript rabbitmq-server, action "start" failed.
● rabbitmq-server.service - RabbitMQ broker
   Loaded: loaded (/lib/systemd/system/rabbitmq-server.service; enabled; vendor preset: enabled)
   Active: failed (Result: exit-code) since Fri 2019-10-11 11:36:43 CST; 6ms ago
  Process: 11003 ExecStop=/usr/lib/rabbitmq/bin/rabbitmqctl stop (code=exited, status=0/SUCCESS)
  Process: 10778 ExecStart=/usr/lib/rabbitmq/bin/rabbitmq-server (code=exited, status=1/FAILURE)
 Main PID: 10778 (code=exited, status=1/FAILURE)
   Status: "Exited."
.....略
Errors were encountered while processing:
 rabbitmq-server
```



5.将插件拷贝到新版本的plugins路径下

在/usr/lib/rabbitmq/lib路径下有2个版本的rabbitmq

```
root@hsq-mq-node2-temp:/usr/lib/rabbitmq/lib# ll
total 16
drwxr-xr-x 4 root root 4096 Oct 11 11:36 ./
drwxr-xr-x 4 root root 4096 Jul  8  2016 ../
drwxr-xr-x 3 root root 4096 Oct 11 11:36 rabbitmq_server-3.6.3/
drwxr-xr-x 6 root root 4096 Oct 11 11:36 rabbitmq_server-3.6.5/
root@hsq-mq-node2-temp:/usr/lib/rabbitmq/lib# pwd
/usr/lib/rabbitmq/lib
```



将备份出来的插件拷贝到3.6.5版本路径下

```
root@hsq-mq-node2-temp:/usr/lib/rabbitmq/lib# mv ~/rabbitmq_delayed_message_exchange-0.0.1.ez  rabbitmq_server-3.6.5/plugins/
```



6.启动rabbitmq-server进程

```
service rabbitmq-server start
```



7.在其他节点重复此文档步骤

---

升级完成后,访问web控制台或者输入命令```rabbitmqctl status```可以看到最新的版本号:

```
RabbitMQ 3.6.5, Erlang 18.3
```

> Centos环境升级完成后Erlang会自动升级到20.0版本