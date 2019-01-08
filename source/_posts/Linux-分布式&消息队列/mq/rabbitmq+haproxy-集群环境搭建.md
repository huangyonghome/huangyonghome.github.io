---
title: rabbitmq+haproxy 集群环境搭建
date: 2018-06-24 12:59:58
tags: MQ
categories: [Linux-分布式&消息队列 ]
comments: true
copyright: true
---



## rabbitmq+haproxy 集群环境搭建



### 实验环境:

4台centos6服务器:

172.16.1.130  rabbitmqnode0 主rabbitmq服务器

172.16.1.131  rabbitmqnode1  节点2

172.16.1.132  rabbitmqnode2  节点3

172.16.1.140 haproxy  负载代理服务器

<!--more-->

1,在三个rabbitmq节点上分别搭建rabbitmq.

2.搭建haproxy服务.....方法略.yum源自带有haproxy工具

yum install haproxy

---



#### 一.rabbitmq安装.

这里只讲解一台服务器的安装.

1.先安装erlang环境

rabbitmq是基于erlang语言开发的.所以需要安装erlang语言包.官网提供了三种安装Erlang的方法:[install Erlang](http://www.rabbitmq.com/install-rpm.html)

1.rabbitmq提供了精简版的erlang包.只安装运行rabbitmq的组件.

2.erlang的源码包

3.EPEL源

这里使用了epel源安装erlang:

```
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-6.noarch.rpm
yum install erlang
```

> 对于centos7的yum源地址为: https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm



2.官网下载rabbitmq-server的RPM包

<http://www.rabbitmq.com/install-rpm.html> 

3.rabbitmq-server需要socat工具,安装socat 

```
wget --no-cache http://www.convirture.com/repos/definitions/rhel/6.x/convirt.repo -O /etc/yum.repos.d/convirt.repo
yum makecache
yum install socat
```

4.安装rabbitmq-server 

```
rpm -ivh rabbitmq-server-3.4.1-1.noarch.rpm
```

5.设置rabbitmq开机启动 

```
chkconfig rabbitmq-server on  
```

6.配置配置文件 

```
cp /usr/share/doc/rabbitmq-server-3.4.1/rabbitmq.config.example /etc/rabbitmq/  
cd /etc/rabbitmq
mv rabbitmq.config.example rabbitmq.config  
```

6.开启用户远程访问 

```
vim /etc/rabbitmq/rabbitmq.config 
取消注释{loopback_users,[]}
```

> note:注意要去掉后面的逗号。 

7.开启web界面管理工具 

```
rabbitmq-plugins enable rabbitmq_management 
```

8.启动rabbitmq-server 

```
  service rabbitmq-server restart  
```

---

**我在启动的时候遇见两个坑** 

**1.ERROR: epmd error for host 172: badarg (unknown POSIX error)** 

解决方案: 

vim /etc/rabbitmq/rabbitmq-env.conf  (这个文件是没有的,需要手动创建)

在文件里面添加这一行：NODENAME=rabbit@localhost，保存

**2.cookie文件权限不对**

```
chown rabbitmq.rabbitmq /var/lib/rabbitmq/.erlang.cookie
chmod 600 /var/lib/rabbitmq/.erlang.cookie
```

9.开放端口 : 15672/5672 



至此.Rabbitmq安装完毕

-----



### 二.集群搭建

1.在三台rabbitmq服务器上分别指定对方的Hostname和ip地址关系

![img](H:/%E6%9C%89%E9%81%93%E4%BA%91%E7%AC%94%E8%AE%B0%E6%96%87%E4%BB%B6/mailhuangyong@163.com/459fe491005c438b954e4633a04ae74a/untitle.png)

> 注意 这里要指定的是本机和其他节点的hostname主机名.

> 我在生产上搭建集群的时候,我指定的格式是:rabbit@hostname.所以在这里踩过很深很深的坑..请详细看另外一篇笔记: 生产环境搭建rabbitmq集群遇到的坑



2.把主节点的Erlang cookie复制到其他两台节点

```
[root@rabbitmqnode0 ~]# scp /var/lib/rabbitmq/.erlang.cookie  172.16.1.131:/var/lib/rabbitmq/
[root@rabbitmqnode0 ~]# scp /var/lib/rabbitmq/.erlang.cookie  172.16.1.132:/var/lib/rabbitmq/
```

3.在其他2个节点上使用-detached运行

```
[root@rabbitmqnode1 rabbitmq]# rabbitmqctl stop
[root@rabbitmqnode1 rabbitmq]# rabbitmq-server -detached
```

(节点2服务器过程略)

4.查看各节点状态

```
rabbitmqctl status
```

5.在其他2个节点上创建并部署集群

```
[root@rabbitmqnode1 rabbitmq]# rabbitmqctl stop_app
[root@rabbitmqnode1 rabbitmq]# rabbitmqctl join_cluster rabbit@rabbitmqnode0  #加入到rabbitmqnode0集群
[root@rabbitmqnode1 rabbitmq]# rabbitmqctl start_app
```

>  我在这里踩到过坑.加入集群报错.重新 rabbitmq-server -detached又正常了,很奇怪



6.查看集群状态

```
[root@rabbitmqCluster]#rabbitmqctl cluster_status
```

集群到此部署完毕.

---



访问主节点页面可以看到rabbitmq集群状态,使用默认的用户密码:guest登录

<http://172.16.1.130:15672/#/>


------



### 三.配置haproxy服务

```
vim /etc/haproxy/haproxy.cfg
###########全局配置#########
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy     # 改变当前工作目录
    stats socket /run/haproxy/admin.sock mode 660 level admin   # 创建监控所用的套接字目录
    pidfile  /var/run/haproxy.pid   # haproxy的pid存放路径,启动进程的用户必须有权限访问此文件
    maxconn  4000                   # 最大连接数，默认4000
    user   haproxy                  # 默认用户
    group   haproxy                 # 默认用户组
    daemon                          # 创建1个进程进入deamon模式运行。此参数要求将运行模式设置为"daemon
    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private
    # Default ciphers to use on SSL-enabled listening sockets.
    # For more information, see ciphers(1SSL). This list is from:
    #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3
###########默认配置#########
defaults
    log global
    mode    http                                # 默认的模式mode { tcp|http|health }，tcp是4层，http是7层，health只会返回OK
    option  httplog                             # 采用http日志格式
    option  dontlognull                         # 启用该项，日志中将不会记录空连接。所谓空连接就是在上游的负载均衡器
                                                # 或者监控系统为了探测该 服务是否存活可用时，需要定期的连接或者获取某
                                                # 一固定的组件或页面，或者探测扫描端口是否在监听或开放等动作被称为空连接；
                                                # 官方文档中标注，如果该服务上游没有其他的负载均衡器的话，建议不要使用
                                                # 该参数，因为互联网上的恶意扫描或其他动作就不会被记录下来
    timeout connect 5000                    # 连接超时时间
    timeout client  50000                   # 客户端连接超时时间
    timeout server  50000                   # 服务器端连接超时时间
    option  httpclose       # 每次请求完毕后主动关闭http通道
    option  httplog         # 日志类别http日志格式
    #option  forwardfor      # 如果后端服务器需要获得客户端真实ip需要配置的参数，可以从Http Header中获得客户端ip  
    option  redispatch      # serverId对应的服务器挂掉后,强制定向到其他健康的服务器
    timeout connect 10000   # default 10 second timeout if a backend is not found
    maxconn     60000       # 最大连接数
    retries     3           # 3次连接失败就认为服务不可用，也可以通过后面设置
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http
####################################################################
listen http_front
        bind 0.0.0.0:1080           #监听端口  
        stats refresh 30s           #统计页面自动刷新时间  
        stats uri /haproxy?stats            #统计页面url  
        stats realm Haproxy Manager #统计页面密码框上提示文本  
        stats auth admin:admin      #统计页面用户名和密码设置  
        #stats hide-version         #隐藏统计页面上HAProxy的版本信息
#####################我把RabbitMQ的管理界面也放在HAProxy后面了###############################
listen rabbitmq_admin
    bind 0.0.0.0:8004
    server node1 172.16.1.130:15672
    server node2 172.16.1.131:15672
    server node3 172.16.1.132:15672
####################################################################
listen rabbitmq_cluster
    bind 0.0.0.0:5672
    option tcplog
    mode tcp
    timeout client  3h
    timeout server  3h
    option          clitcpka
    balance roundrobin      #负载均衡算法（#banlance roundrobin 轮询，balance source 保存session值，支持static-rr，leastconn，first，uri等参数）
    #balance url_param userid
    #balance url_param session_id check_post 64
    #balance hdr(User-Agent)
    #balance hdr(host)
    #balance hdr(Host) use_domain_only
    #balance rdp-cookie
    #balance leastconn
    #balance source //ip
    server   node1 172.16.1.130:5672 check inter 5s rise 2 fall 3   #check inter 2000 是检测心跳频率，rise 2是2次正确认为服务器可用，fall 3是3次失败认为服务器不可用
    server   node2 172.16.1.131:5672 check inter 5s rise 2 fall 3
    server   node3 172.16.1.132:5672 check inter 5s rise 2 fall 3
```



启动haproxy

```
service haproxy start
```



浏览器输入:http://172.16.1.140:1080/haproxy?stats  ---用户名和密码admin:admin


[http://172.16.1.140:8004](http://172.16.1.140:8004/)  ---用户名密码:guest

可以看到和rabbitmq主节点一样的页面




**至此haproxy+rabbitmq集群成功搭建完成**

------

### 四.代码测试:

在python上编写producer和consumer代码.连接到Haproxy.端口默认5672不用指定.

发送5条测试消息.



**可以看到成功接收到5条消息**


 队列名为hello


------

**模拟一个节点down掉:**

```
[root@rabbitmqnode2 ~]# rabbitmqctl stop_app
Stopping rabbit application on node rabbit@rabbitmqnode2 ...
```



haproxy和rabbitmq显示一个节点不可用:





节点启动:

```
[root@rabbitmqnode2 ~]# rabbitmqctl start_app
Starting node rabbit@rabbitmqnode2 ...
completed with 3 plugins.
```



回归正常:



---

