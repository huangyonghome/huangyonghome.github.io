---
title: 使用supervisor托管elasticsearch进程
date: 2018-06-15 15:59:58
tags: supervisor
categories: Linux-Service
comments: true
copyright: true
---



# 使用supervisor托管elasticsearch进程



## 背景

线上出现过一次ES集群故障:[记一次线上ES集群故障](https://tower.im/teams/682393/documents/81875/?fullscreen=false)
此文主要介绍ES集群健壮性的整改措施.主要在以下几个方面:

1. 新增一台ES集群Master节点,使节点数达到3个,从而可以容忍一个master节点故障
2. 使用supervisor托管elasticsearch进程
3. 使用systemd托管supervisord.使supervisord开机自动启动

<!--more-->

---
## 详细措施

1.在supervisor的conf.d的目录下新增elasticsearch.conf配置文件.内容如下

```
[program:elasticsearch]
#启动命令,前台启动,不需要-d参数
command=/data/app/elasticsearch-2.4.6/bin/elasticsearch
autorestart=true
autostart=true
stderr_logfile=/data/logs/elasticsearch/hsq_elasticsearch.error.log
stdout_logfile=/data/logs/elasticsearch/hsq_elasticsearch.log
#JAVA_HOME的路径
environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.222.b10-0.el7_6.x86_64/jre
#elasticsearch的启动内存堆栈大小.这个环境变量可以在/etc/profile中定义.但是supervisord无法识别shell终端的环境变量
environment=ES_HEAP_SIZE=24g
#最大文件打开数,同样,最好是在这里的配置文件定义一下.
environment=MAX_OPEN_FILES=102400
#启动用户.注意.elasticsearch不能用root启动
user=work
#进程终止信号
stopsignal=INT
#进程重启间隔
startsecs=10
```

以上就是elasticsearch的配置文件.但是经过我测试,在部分服务器上,ES启动后无法使用预期的ES_HEAP_SIZE变量.所以我选择是在elasticsearch的启动文件中直接写死.

在/data/app/elasticsearch-2.4.6/bin/elasticsearch/bin/elasticsearch.in.sh启动文件中的如下位置新增一行

```
[root@hsq-es-node3 ~]# cat /data/app/elasticsearch-2.4.6/bin/elasticsearch.in.sh

#这里写死ES_HEAP_SIZE内存数
ES_HEAP_SIZE=24g

if [ "x$ES_MIN_MEM" = "x" ]; then
    ES_MIN_MEM=256m
fi
if [ "x$ES_MAX_MEM" = "x" ]; then
    ES_MAX_MEM=1g
fi
if [ "x$ES_HEAP_SIZE" != "x" ]; then
    ES_MIN_MEM=$ES_HEAP_SIZE
    ES_MAX_MEM=$ES_HEAP_SIZE
fi
```

2. 使用systemd托管supervisord实现开机自启

在/usr/lib/systemd/system/路径下新增supervisord.service文件.内容如下

```
[Unit]
Description=Supervisor daemon

[Service]
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisord/supervisord.conf
ExecStop=/usr/bin/supervisorctl -c /etc/supervisord/supervisord.conf  shutdown
ExecReload=/usr/bin/supervisorctl -c /etc/supervisord/supervisord.conf reload
KillMode=process
Restart=on-failure
RestartSec=42s

[Install]
WantedBy=multi-user.target
```

加入开机启动

```
systemctl enable supervisord

```

3. 手动kill掉正在运行中的supervisord和elasticsearch进程.

> 注意观察当前是否有其他重要进程由supervisor托管

然后使用```systemctl start supervisord```启动supervisor.此时elasticsearch会自动启动

----

## ES集群新增master节点

编辑elasticsearch.yaml文件.修改如下字段

```
node.master: true
node.data: false
node.client: false

```
此时该节点启动后,就只做为master节点运行.

在其他ES节点上修改配置中的以下配置,将所有节点的IP和端口添加进列表:

```
discovery.zen.ping.unicast.hosts: ["10.111.30.202:9300","10.111.30.207:9300","10.111.30.206:9300","10.111.30.193:9300","10.111.30.197:9300"]
```