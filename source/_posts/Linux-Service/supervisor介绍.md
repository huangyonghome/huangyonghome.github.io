---
title: supervisor介绍
date: 2018-06-15 15:59:58
tags:  Linux,supervisor
categories: [Linux-Service ]
comments: true
copyright: true
---



## supervisor介绍

官网地址:[supervisor]([http://supervisord.org](http://supervisord.org/) )

#### 一.概述

supervisor是一个client/server架构,允许用户监控和控制一系列的Linux操作系统进程.

#### 二.安装

supervisor是基于Python2开发的.所以安装很简单.只需要以下步骤

<!--more-->

1.安装python2的Pip2工具

```
yum search python2-pip 
```



2.用pip工具安装supervisor

```
pip install supervisor
```

3.生成一份配置文件

```
echo_supervisord_conf >配置文件路径
```

> note: supervior工具不支持python3

---

三.配置文件

supervisord.conf是supervisor的配置文件.该配置文件是Windows-INI类型定义的.

分号;开头的为注释行.每个应用以中括号[]开头.在下面以KEY/VALUE键值对定义参数.

以下列举了一些重要参数配置. 

> 详细配置请参考官方文档[supervisor配置](http://supervisord.org/configuration.html)

```
[unix_http_server]
侦听一个socket接口.
[inet_http_server]
侦听一个TCP接口

[supervisord] ----这部分是supervisor的全局配置

logfile : supervisor的日志文件
logfile_maxbytes:单个日志文件的最大容量.单位支持KB,MB,GB.设置为0表示无限制 .默认50M
logfile_backups: 轮替保留多久.0表示不限制 .默认10
loglevel: 记录日志级别.默认info
pidfile:supervisor本身的pid文件
nodaemon: 如果为true则supervisord会在前台运行.默认false
minfds: 最小文件描述符.默认1024
user:指定supervisord切换到哪个用户下运行

[supervisorctl] -----supervisorctl的shell交互接口的配置

URL:访问supervisord服务的URL接口. e.g. http://localhost:9001. For UNIX domain sockets, use unix:///absolute/path/to/file.sock.
prompt:定义一个supervisorctl的shell交互界面的提示符.默认是supervisor
history_file : 如果定义了history_file的文件路径.那么你每次使用supervisorctl的命令会被保存在这个文件中.类似于linux系统的.bash_history文件.默认没有配置

[program:X] -----定义一个具体的应用,指示supervisor如何对该控制该应用.x表示一个应用名
 
command: 定义应用程序启动文件的绝对路径.支持程序启动参数选项
priority:应用程序启动或者关闭的优先级.当supervisorctl start/stop all时,将按照此优先级顺序启动或者关闭应用程序 
autostart: 设置为true.supervisord启动时,会自动启动该应用
startsecs:应用程序启动后需要停留在running状态的秒数,当启动后保持在running状态达到此值后,supervisor才会认为是成功启动.设置为0表示不需要停留在running状态.此值用来检测应用程序是否成功启动
startretries:当应用程序启动失败时,supervisord允许尝试重新启动的次数.如果启动N次后还是失败,则会放弃启动,并且将此应用程序状态设置为FATAL状态
autorestart:如果一个应用程序退出后(比如被手动Kill掉了),是否需要自动重启该应用程序.设置为false则不会重新启动,设置为true则无条件的重新启动.
stopsignal: 使用supervisor stop 一个应用时,可以指定kill信号:有 TERM,HUP,KILL,QUIT,INT,USER1,USER2
user: 运行应用程序的用户名
stdout_logfile: 应用程序的输出日志文件路径.Note:需要手动创建日志文件目录,否则目录不存在时,无法启动supervisord
stdout_logfile_maxbytes: stdout_logfile定义的日志文件最大字节数.单位支持KB,MB,GB.设置为0表示无限制 .默认50MB
studout_logfile_backups:轮替保留多久.0表示不限制 .默认10
redirect_stderr:如果设置为true.则把错误日志重定向到studout_logfile.默认为false
stderr_logfile: 应用程序的错误日志文件路径.其他和stdout_logfile相同,
stderr_logfile_maxbytes: 
stderr_logfile_backups:
	
[include] ----包含某个目录下的所有配置文件

files:指定目录下的配置文件


[group:X]---将多个不同的应用程序定义为一个组
```

以下列举了nginx的进程管理配置参考:

```
[program:nginx]
command=/usr/sbin/nginx
numprocs=1
startsecs = 3
startretries = 5
user=root
stdout_logfile = /var/logs/nginx/nginx.log
stderr_logfile = /var/logs/nginx/nginx.log
redirect_stderr = true
autostart=true
autorestart=true
```

---

#### 四.使用supervisor

使用supervisor管理一个应用,必须将程序写入supervisord.conf文件. 

格式如下:

```
[program:应用名]
command=应用启动程序文件
..... 
```

定义完supervisord.conf配置文件.就可以启动supervisor了.. 

启动方式: 

```
$BINDIR/supervisord  

启动参数: 

-c 指定一个配置文件路径
-n 前台执行.有助于查看启动过程中输出信息拍错
-u 以指定用户运行
```

>note:启动时最好-c 指定一个配置文件的绝对路径.因为supervisor默认在当前工作目录寻找配置文件. 

---

#### 五.运行supervisorctl 

启动方式:  

```
1.$BINDIR/supervisorctl -c /etc/supervisord/supervisord.conf #这种方式也启动一个shell接口.进入supervisorctl交互界面
2.$BINDIR/supervisorctl -c /etc/supervisord/supervisord.conf command  #这种方式只对supervisorctl进行一次调用.成功执行后retrun 0 .不会启动一个交互shell界面 
```

> 注意:supervisorctl启动也需要指定参数.不然可能报错误 



**supervisorctl 动作介绍:** 

- add <name> 添加一个刚刚添加到supervisord.conf配置文件中的应用.

- remove <name>和add相反

- update重新加载配置文件.并且会自动add/remove应用(如果配置文件有修改)

- clear <name>清除一个程序的日志文件

- clear all清除所有程序的日志文件

- fg <应用>将某个应用调到前台运行.按CTRL+C退出

- pid <name> 获得某个应用的PID

- pid all获得所有应用的pid

- reread 和Update相似,但是不会自动add/remove

- restart <name> <name>重启一个或多个应用,

  > 注意:restart并不会重载supervisord.conf配置文件

- restart all 和restart 类似start <name> <name> ...启动一个或多个应用

- start all 启动所有应用

- status获得所有应用的状态

- status <name> <name>获得一个或多个应用状态

- stop <name> <name>停止一个或多个应用

- stop all停止所有 

---



##### 最后 **关于autorestart字段:**

autorestart还有个值是unexpected.并且默认是unexpected

如果设置为unexpected.那么要结合另外一个属性一起使用:exitcodes . exitcodes属性的值是以逗号为分割的多个数字.

如果设置为unexpected.那么当程序关闭时.程序的退出码(exit code .程序异常关闭,或者被kill,exit code会不一样)如果不在exitcodes定义中内.那么supervisor会autorestart自动重启该应用程序.

如果程序被关闭时的退出码没有被exitcodes指定,那么supervisor会自动重启这个程序.默认值为0,2

如果程序的退出码在exitcodes指定中.那么不会自动重启该程序.

白话解释就是:明确指定的退出状态码,supervisor不会自动启动该程序..如果程序的退出码没有手动指定(也就是意料之外的unexpected),那么supervisor会自动重启该程序 



比如下面这个例子,supervisor反复的启动elasticsearch..注意观察elasticsearch的PID一直在变. : 

{% qnimg linux/supervisor.png %}



 查看supervisor的日志: 

elasticsearch一直在不断的启动,异常退出,启动.并且看到elasticsearch的退出码为1 

{% qnimg linux/supervisor1.png %}



查看elasticsearch的日志: 

可以看到elasticsearch一直在退出.然后被supervisor自动重启 

{% qnimg linux/supervisor2.png %}



修改supervisord.conf配置文件中的elasticsearch配置文件

将autostart=true改成unexpected,

并且加入exitcodes属性.值设为1. 

重启supervisor进程

> note:重启supervisord进程前.需要kill掉所有supervisord管理的应用程序 

重载supervisor配置文件,可以发现supervisor并没有自动重启该应用 

{% qnimg linux/supervisor3.png %}

---









