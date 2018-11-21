---
title: kong+casssandra集群环境部署
date: 2018-11-21 5:22:58
tags: kong
categories: [Linux-分布式&消息队列 ]
comments: true
copyright: true
---

## kong+casssandra集群环境部署

### kong简介
Kong是Mashape开源的一款API网关，起初是用来管理 Mashape 公司15000个微服务的，后来在2015年开源,现在已经在很多创业公司、大型企业和政府机构中广泛使用。基于nginx,Lua和Cassandra或PostgreSQL，支持分布式操作，有很强的可移植性和可扩展性。可以在任何一种基础设施上运行,作为应用和API之间的中间层，加上众多功能强大的插件，可以实现认证授权、访问控制等功能。并且提供易于使用的RESTful API来操作和配置系统。

有关kong的详细介绍请参考官网.

--

### cassandra简介
Cassandra 是一个来自 Apache 的分布式数据库，具有高度可扩展性，可用于管理大量的结构化数据。它提供了高可用性，没有单点故障。kong支持PostgreSQL或者Cassandra两种数据库.这里我们选择了cassandra.

有关cassandra的详细介绍和使用方法.请参考官网

--

### 集群架构

* **kong cluster**

kong 集群并不意味着客户端请求将会负载均衡到kong集群中的每个节点上，kong集群并不是开箱即用，仍然需要在kong集群多节点上层搭建负载均衡，以便分发请求。 一个kong集群只是意味着集群内的节点，都共享同样的配置。

有关Kong cluster集群的详细介绍请参考官网:[Kong cluser document](https://docs.konghq.com/0.14.x/clustering/)

为了提高冗余性和健壮性.我们对kong的每个环节都进行了冗余设计.一个基本的kong集群架构大概如下图所示:

![](http://pabkmteb4.bkt.clouddn.com/kong-flow.png)

--

### 部署步骤

**环境**: 

阿里云ECS Centos7.4操作系统  
Kong: 0.14 最新版
Cassandra: 3.11 最新版

--

#### 安装Cassandra

安装方式官网参考: [Installing Cassandra](http://cassandra.apache.org/doc/latest/getting_started/installing.html#installation-from-binary-tarball-files)

**安装前提条件**

1.安装JDK 8版本  
2.安装2.7以上版本的Python.(cassandra管理工具:cqlsh 需要python2.7以上环境)

**安装步骤**

1.下载cassandra二进制文件.

```
wget http://mirrors.tuna.tsinghua.edu.cn/apache/cassandra/3.11.3/apache-cassandra-3.11.3-bin.tar.gz
```

2.将cassandra目录添加进环境变量.用work用户运行cassandra

```
sudo mkdir /usr/local/cassandra
sudo chown -R work:work /usr/local/cassandra
sudo tar -xvf apache-cassandra-3.11.3-bin.tar.gz -C /usr/local/cassandra

cd /usr/local/cassandra
sudo mv apache-cassandra-3.11.3/* .
sudo rm apache-cassandra-3.11.3/ -rf
```

添加进环境变量

```
sudo vim /etc/profile

export JAVA_HOME=/usr/local/java
export CASSANDRA_HOME=/usr/local/cassandra
export PATH=$PATH:$JAVA_HOME/bin:$CASSANDRA_HOME/bin
```
--

#### 配置Cassandra,以及cassandra集群

1.编辑cassandra的cassandra.yml配置文件.修改下列配置:

```
[work@kong-node1 kong]$ vim /usr/local/cassandra/conf/cassandra.yaml

#定义Cassandra集群名
cluster_name: 'dwd_cassandra'
#定义hints路径.可以使用默认路径
hints_directory: /data/cassandra/hints

#采用密码方式连接数据库.默认情况下不需要任何用户密码就可以登录数据库
authenticator: PasswordAuthenticator

#定义数据库文件路径.可以使用默认/var/lib路径
data_file_directories:
      - /data/cassandra

#定义commit日志路径.可以使用默认路径
commitlog_directory: /data/cassandra/commitlog

#缓存文件路径
saved_caches_directory: /data/cassandra/saved_caches

#关键配置,定义集群种子服务器地址.这里定义服务器的内网地址.不能使用0.0.0.0或者127的本机地址,可以加入多个集群节点的地址,IP地址之间用逗号分隔
- seeds: "10.25.87.159"

#listen地址
listen_address: 10.25.87.159

#rpc地址
rpc_address: 10.25.87.159

```

2.创建刚才定义的路径目录

```
sudo mkdir -pv /data/cassandra 
sudo chown -R work.work /data/
```

3.启动cassandra.直接在命令行执行cassandra

```
[work@kong-node1 kong]$ cassandra 

或者

[work@kong-node1 kong]$ /usr/local/cassandra/bin/cassandra

```

4. 使用cqlsh工具登陆cassandra数据库.创建cassandra用户密码,以及创建键空间

```
#注意由于cassandra只侦听了内网的地址,因此要指定IP地址.
#默认账号密码都是cassandra
[work@kong-node1 kong]$ cqlsh 10.25.87.159  -ucassandra -pcassandra 

#创建一个kong用户.并且为超级用户
cassandra@cqlsh> create user kong with password 'kong' superuser;

#创建一个keyspace.命名为kong
cassandra@cqlsh> CREATE KEYSPACE kong WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1};

cassandra@cqlsh> exit
```

5.删除自带的cassandra用户

```
[work@kong-node1 cassandra]$ cqlsh 10.25.87.159  -ukong -pkong
kong@cqlsh> desc kong;

CREATE KEYSPACE kong WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'}  AND durable_writes = true;

kong@cqlsh> drop user cassandra;
```
--

#### 安装Kong

安装方法可以参考官网:[Install Kong](https://docs.konghq.com/install/centos/?_ga=2.110797315.728319704.1539597667-917309945.1539077269#packages)

1.下载,安装rpm安装包

```
wget -O kong-community-edition-0.14.1.el7.noarch.rpm  https://bintray.com/kong/kong-community-edition-rpm/download_file?file_path=centos/7/kong-community-edition-0.14.1.el7.noarch.rpm

sudo yum install epel-release
sudo yum install kong-community-edition-0.14.1.el7.noarch.rpm
```

2.修改配置文件

```
cd /etc/kong
sudo cp kong.conf.default kong.conf

sudo vim kong.conf

#修改日志文件路径
prefix = /data/kong/

#由于磁盘空间有限,关闭kong的代理日志.后端真实服务器会记录nginx访问日志
proxy_access_log = off
proxy_error_log = off

#在所有地址侦听管理端口,当然只侦听127地址会更安全.
admin_listen = 0.0.0.0:8001, 0.0.0.0:8444 ssl

#指定使用cassandra数据库
database = cassandra
#数据库地址,端口
cassandra_contact_points = 10.25.87.159
cassandra_port = 9042

#上文定义的cassandra数据库的用户密码和键空间
cassandra_keyspace = kong
cassandra_username = kong
cassandra_password = kong

#kong官方建议的cassandra一致性机制
cassandra_consistency = QUORUM

#以下是集群的数据库和缓存方面的配置.详细介绍请参考官网
db_update_frequency = 5
db_update_propagation = 2

```

3.创建kong目录

```
mkdir /data/kong
```

4.准备启动工作

```
kong migrations up -c /etc/kong/kong.conf
```
5.启动kong

```
kong start -c /etc/kong/kong.conf
```

查看端口.可以看到cassandra和kong的侦听端口都已经成功启动

```
work@kong-node1 kong]$ netstat -tulnp
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      29342/nginx: master
tcp        0      0 0.0.0.0:8444            0.0.0.0:*               LISTEN      29342/nginx: master
tcp        0      0 127.0.0.1:7199          0.0.0.0:*               LISTEN      28598/java
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      29342/nginx: master
tcp        0      0 0.0.0.0:8001            0.0.0.0:*               LISTEN      29342/nginx: master
tcp        0      0 127.0.0.1:35503         0.0.0.0:*               LISTEN      28598/java
tcp        0      0 10.25.87.159:9042       0.0.0.0:*               LISTEN      28598/java
tcp        0      0 10.25.87.159:7000       0.0.0.0:*               LISTEN      28598/java

```

---


### 部署另外一台kong和cassandra

今天在阿里云镜像了kong-node1的服务器.新的服务器名字为kong-node2.  
软件已经安装,只需要修改部分配置

##### 修改node1和node2上的cassandra配置

```
在node1和node2上:
[work@kong-node2 ~]$ vim /usr/local/cassandra/conf/cassandra.yaml
#修改seeds配置.添加2台服务器的内网IP地址
- seeds: "10.25.87.159, 10.80.229.244" 

在node2上修改侦听地址
[work@kong-node2 ~]$ vim /usr/local/cassandra/conf/cassandra.yaml
#将下列地址改成node2内网地址
listen_address: 10.80.229.244
rpc_address: 10.80.229.244
```
##### 在node2上修改kong的配置文件:
```
work@kong-node2 ~]$ vim /etc/kong/kong.conf
#连接本机的cassandra数据库地址
cassandra_contact_points = 10.80.229.244
```

> note:千万不要启动node2上的cassandra.因为node2是从node1镜像过去的.所以数据库的token是一模一样的.


##### 在node2上删除cassandra的数据库.这一步非常重要.因为这是从node1镜像过来的.所以node2上的数据库是node1的数据

```
rm -rf /data/cassandra/*
```

##### 启动node2上的数据库

直接在命令行执行:cassandra

##### 查看cassandra的单台服务器状态

```
[work@kong-node1 bin]$ nodetool info
ID                     : 4fe1df37-e69e-4a25-acdc-4b2d73a92225
Gossip active          : true
Thrift active          : false
Native Transport active: true
Load                   : 522.87 KiB
Generation No          : 1540372001
Uptime (seconds)       : 63895
Heap Memory (MB)       : 404.89 / 1004.00
Off Heap Memory (MB)   : 0.00
Data Center            : datacenter1
Rack                   : rack1
Exceptions             : 0
Key Cache              : entries 59, size 5.03 KiB, capacity 50 MiB, 7540 hits, 7911 requests, 0.953 recent hit rate, 14400 save period in seconds
Row Cache              : entries 0, size 0 bytes, capacity 0 bytes, 0 hits, 0 requests, NaN recent hit rate, 0 save period in seconds
Counter Cache          : entries 0, size 0 bytes, capacity 25 MiB, 0 hits, 0 requests, NaN recent hit rate, 7200 save period in seconds
Chunk Cache            : entries 28, size 1.75 MiB, capacity 219 MiB, 1237 misses, 26133 requests, 0.953 recent hit rate, NaN microseconds miss latency
Percent Repaired       : 100.0%
Token                  : (invoke with -T/--tokens to see all 256 tokens)
```
##### 查看cassandra集群状态.可以看到集群中2台服务器

```
[work@kong-node1 bin]$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.80.229.244  339.93 KiB  256          51.3%             04a75f63-be99-4f3e-93ff-937bbe9656d8  rack1
UN  10.25.87.159   522.87 KiB  256          48.7%             4fe1df37-e69e-4a25-acdc-4b2d73a92225  rack1

```

---

#### 以下是我踩过的坑.在没有删除node2上的数据库文件情况下,直接启动cassnadra.


启动node2的cassandra后.发现集群无法正常启动.使用cassandra自带的nodetool工具可以查看集群状态.这里只有自己一台服务器

```
[work@kong-node2 ~]$ nodetool status
Datacenter: datacenter1
=======================
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address        Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.80.229.244  600.05 KiB  256          100.0%            4fe1df37-e69e-4a25-acdc-4b2d73a92225  rack1

```
查看cassandra启动日志,发现日志提示和node1有一样的token:

```
less /usr/local/cassandra/logs/system.log

INFO  [GossipStage:1] 2018-10-24 17:07:26,874 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token -9066137612411979055.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,874 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token -912539082610246005.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,874 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token -9125687604150710607.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,874 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token -9186325188411815558.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,874 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token -934168442605847346.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,874 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token -937629522304513228.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,874 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token -983284835358960159.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,875 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token 1111859401021864246.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,875 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token 1185525604491731552.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,875 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token 1209704333924286496.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,875 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token 1243859262038298713.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,875 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token 1284321765579584761.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,875 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token 1472069791929520463.  Ignoring /10.25.87.159
INFO  [GossipStage:1] 2018-10-24 17:07:26,875 StorageService.java:2386 - Nodes /10.25.87.159 and /10.80.229.244 have the same token 1479257042759500258.  Ignoring /10.25.87.159
```

不仅如此,在node1上启动kong,提示cassandra数据库验证失败.以及提示kong需要migrations up(只需要在第一次启动kong时才需要migratios up):

```
[work@kong-node1 bin]$ kong restart -c /etc/kong/kong.conf
Error: /usr/local/share/lua/5.1/kong/cmd/start.lua:37: [cassandra error] the current database schema does not match this version of Kong. Please run `kong migrations up` to update/initialize the database schema. Be aware that Kong migrations should only run from a single node, and that nodes running migrations concurrently will conflict with each other and might corrupt your database schema!
```

kong migrations失败

```

 work@kong-node1 bin]$ kong migrations up -c /etc/kong/kong.conf
migrating core for keyspace kong
core migrated up to: 2015-01-12-175310_skeleton
Error: [cassandra error] Error during migration 2015-01-12-175310_init_schema: [Invalid] Undefined column name request_host
```

##### 解决方法

启动node1和node2的cassandra


1.在node1和node2上drop kong的键空间

```
[work@kong-node1 bin]$ cqlsh 10.25.87.159 -ukong -pkong
Connected to dwd_cassandra at 10.25.87.159:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
kong@cqlsh> drop keyspace kong;
kong@cqlsh> exit

[work@kong-node2 ~]$ cqlsh 10.80.229.244 -ukong -pkong
Connected to dwd_cassandra at 10.80.229.244:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
kong@cqlsh> drop KEYSPACE kong;
ConfigurationException: Cannot drop non existing keyspace 'kong'.
kong@cqlsh> exit
```
2.删除kong的键空间目录

```

[work@kong-node1 bin]$ rm -rf  /data/cassandra/kong/
[work@kong-node1 bin]$ ll /data/cassandra/kong/
ls: cannot access /data/cassandra/kong/: No such file or directory


[work@kong-node2 ~]$ rm -rf /data/cassandra/kong/
[work@kong-node2 ~]$ ll /data/cassandra/kong/
ls: cannot access /data/cassandra/kong/: No such file or directory
```


3.在node1上创建kong键空间.创建完毕后,应该会同步到node2

```
[work@kong-node1 bin]$ cqlsh 10.25.87.159 -ukong -pkong
Connected to dwd_cassandra at 10.25.87.159:9042.
[cqlsh 5.0.1 | Cassandra 3.11.3 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
kong@cqlsh> CREATE KEYSPACE kong WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 1};
kong@cqlsh> exit
```
4.在node1上执行 kong migrations up ,执行完后同样会同步到node2

```
work@kong-node1 bin]$ kong migrations up -c /etc/kong/kong.conf
migrating core for keyspace kong
...

66 migrations ran
waiting for Cassandra schema consensus (10000ms timeout)...
Cassandra schema consensus: reached
```

5.在node1和node2上启动kong

```
[work@kong-node1 bin]$ kong restart -c /etc/kong/kong.conf

[work@kong-node2 ~]$ kong start -c /etc/kong/kong.conf
Kong started
```
---

### 搭建kong-dashboard

kong-dashboard是管理kong各个组件(serveice,route,plugin,upstream,consumer)的可视化UI管理工具.在增删改查各个组件的配置时非常方便.

现在kong-dashboard也支持到了最新版的kong和kong的最新组件.

kong-dashboard的github参考:[kong-dashboar](https://github.com/PGBI/kong-dashboard)

---

1.确保需要有node.js环境.如果没有npm工具,必须先安装nodejs

```
[work@DWD-BETA kong]$ npm -v
5.6.0
[work@DWD-BETA kong]$ node -v
v8.10.0
```

2.root用户执行下列安装命令

```
[root@DWD-BETA ~]# npm install -g kong-dashboard
/usr/local/src/nodejs/bin/kong-dashboard -> /usr/local/src/nodejs/lib/node_modules/kong-dashboard/bin/kong-dashboard.js
+ kong-dashboard@3.5.0
added 184 packages in 28.8s
```
3.启动kong-dashboard.启动方式有以下几种

```
# Start Kong Dashboard
kong-dashboard start --kong-url http://kong:8001

# Start Kong Dashboard on a custom port
kong-dashboard start \
  --kong-url http://kong:8001 \
  --port [port]

# Start Kong Dashboard with basic auth
kong-dashboard start \
  --kong-url http://kong:8001 \
  --basic-auth user1=password1 user2=password2

# See full list of start options
kong-dashboard start --help
```

但是kong-dashboard是前台启动,没有deamnize模式.所以将kong-dashboard加入到supervisor进程管理

```
[root@DWD-BETA ~]# vim /etc/supervisor/conf.d/kong-dashboard.conf

[program:kong-dashboard]
command=/usr/local/src/nodejs/bin/kong-dashboard start --kong-url http://localhost:8001
numprocs=1
user=work
stdout_logfile = /data/logs/kong/kong-dashboard.log
redirect_stderr = true
autostart=true
autorestart=true

```
修改权限

```
[root@DWD-BETA ~]# chown -R work.work /etc/supervisor/conf.d/
```
更新supervisor配置文件

```
[root@DWD-BETA ~]# supervisorctl update kong-dashboard
kong-dashboard: added process group
```

但是由于我这台服务器上8080端口已经被使用,所以启动kong-dashboard报错:

```
work@DWD-BETA kong]$ /usr/local/src/nodejs/bin/kong-dashboard start --kong-url http://localhost:8001
Connecting to Kong on http://localhost:8001 ...
Connected to Kong on http://localhost:8001.
Kong version is 0.14.1
Starting Kong Dashboard on port 8080
events.js:183
      throw er; // Unhandled 'error' event
      ^

Error: listen EADDRINUSE :::8080
    at Object._errnoException (util.js:1022:11)
    at _exceptionWithHostPort (util.js:1044:20)
    at Server.setupListenHandle [as _listen2] (net.js:1367:14)
    at listenInCluster (net.js:1408:12)
    at Server.listen (net.js:1492:7)
    at Application.listen (/usr/local/src/nodejs/lib/node_modules/kong-dashboard/node_modules/koa/lib/application.js:65:19)
    at Server.start (/usr/local/src/nodejs/lib/node_modules/kong-dashboard/lib/server.js:32:9)
    at startKongDashboard (/usr/local/src/nodejs/lib/node_modules/kong-dashboard/bin/kong-dashboard.js:189:10)
    at request.get.then.then (/usr/local/src/nodejs/lib/node_modules/kong-dashboard/bin/kong-dashboard.js:178:5)
    at <anonymous>
```

8080端口被jenkins占用了

```
[work@DWD-BETA kong]$ netstat -tulnp | grep 8080
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      17955/java
```

更换kong-dashboard端口为8081

```
[program:kong-dashboard]
command=/usr/local/src/nodejs/bin/kong-dashboard  start --kong-url http://localhost:8001 --port 8081
numprocs=1
user=work
stdout_logfile = /data/logs/kong/kong-dashboard.log
redirect_stderr = true
autostart=true
autorestart=true
```
更新supervisor后,仍然无法启动

```
[work@DWD-BETA kong]$ supervisorctl update kong-dashboard
kong-dashboard: stopped
kong-dashboard: updated process group
[work@DWD-BETA kong]$ supervisorctl status kong-dashboard
kong-dashboard                   BACKOFF   Exited too quickly (process log may have details)

```

手动启动正常

```
[work@DWD-BETA kong]$ /usr/local/src/nodejs/bin/kong-dashboard  start --kong-url http://localhost:8001 --port 8081
Connecting to Kong on http://localhost:8001 ...
Connected to Kong on http://localhost:8001.
Kong version is 0.14.1
Starting Kong Dashboard on port 8081
Kong Dashboard has started on port 8081
^C
```

查看supervisor启动日志文件:

```
[work@DWD-BETA kong]$ less /data/logs/kong/kong-dashboard.log
/usr/bin/env: node: No such file or directory
/usr/bin/env: node: No such file or directory
/usr/bin/env: node: No such file or directory
/usr/bin/env: node: No such file or directory
```

网上查找解决方案.说是要链接以下文件

```
sudo ln -s /usr/bin/nodejs /usr/bin/node
```

但是这台服务器上nodejs路径不同,所以执行以下命令

```
[work@DWD-BETA kong]$ sudo ln -s /usr/local/src/nodejs /usr/bin/node

```
仍然无法启动,提示

```
/usr/bin/env: node: Permission denied
/usr/bin/env: node: Permission denied
/usr/bin/env: node: Permission denied
/usr/bin/env: node: Permission denied
/usr/bin/env: node: Permission denied
/usr/bin/env: node: Permission denied
```
修改kong-dashboard启动用户为root.仍然无法启动.

```
[program:kong-dashboard]
command=/usr/local/src/nodejs/bin/kong-dashboard  start --kong-url http://127.0.0.1:8001 --port 8081
numprocs=1
user=root
stdout_logfile = /data/logs/kong/kong-dashboard.log
redirect_stderr = true
autostart=true
autorestart=true
```

---

操作失误在创建软件的时候,删除了nodejs源目录.重新安装了npm和kong-dashboard

```
解压nodejs包到:/usr/local/node/

#设置环境变量: 
[root@DWD-BETA ~]# cat /etc/profile | tail -2
export NODEJS_HOME=/usr/local/node/
export PATH=$PATH:$NODEJS_HOME/bin

[root@DWD-BETA ~]# node -v
v8.10.0
[root@DWD-BETA ~]# npm -v
5.6.0
```

创建链接

```
[root@DWD-BETA ~]# ln -s /usr/local/node/bin/node /usr/bin/node
root@DWD-BETA ~]# ll /usr/bin/node -d
lrwxrwxrwx 1 root root 24 Nov  3 11:25 /usr/bin/node -> /usr/local/node/bin/node
```

修改supervisor配置文件中的命令路径

```
[program:kong-dashboard]
command=/usr/local/node/bin/kong-dashboard  start --kong-url http://127.0.0.1:8001 --port 8081
numprocs=1
priority=3
user=root
stdout_logfile = /data/logs/kong/kong-dashboard.log
redirect_stderr = true
autostart=true
autorestart=true

```

启动kong-dashboard

```
[root@DWD-BETA ~]# supervisorctl update kong-dashboard
kong-dashboard: stopped
kong-dashboard: updated process group

[root@DWD-BETA ~]# supervisorctl status kong-dashboard
kong-dashboard                   RUNNING   pid 16635, uptime 0:05:02
```

启动完成

```
[root@DWD-BETA ~]# netstat -tulnp | grep 8081
tcp6       0      0 :::8081                 :::*                    LISTEN      16635/node
```

---

### 将cassandra加入到supervisor进程管理

cassandra加入supervisor进程托管遇到不少问题.踩过以下2个坑:

1.启动报错,提示需要更高级版本的java.

```
Cassandra 3.0 and later require Java 8u40 or later.
Cassandra 3.0 and later require Java 8u40 or later.
Cassandra 3.0 and later require Java 8u40 or later.
Cassandra 3.0 and later require Java 8u40 or later.
Cassandra 3.0 and later require Java 8u40 or later.
Cassandra 3.0 and later require Java 8u40 or later.
Cassandra 3.0 and later require Java 8u40 or later.
```

我的解决方案:

* 在cassandra的supervisor配置文件中加入环境变量:

```
environment=JAVA_HOME="/usr/local/java"  #这样cassandra会识别用户自定义安装的Java.
```

* 配置软链

```
sudo ln -s /usr/local/java/bin/java /usr/bin/java
```

2.仍然无法启动,因为命令行是以daemon方式启动.在cassandra的supervisor配置文件中的启动参数加入-f选项.

最终的cassandra配置文件如下:

```
[program:cassandra]
command=/usr/local/cassandra/bin/cassandra -f
directory=/usr/local/cassandra/
environment=JAVA_HOME="/usr/local/java"
enviroment=PATH="$JAVA_HOME/bin:$PATH"
priority=0
numprocs=1
user=work
stdout_logfile = /data/logs/cassandra/cassandra_supervisor.log
redirect_stderr = true
autostart=true
autorestart=true
```

---

### 将kong加入到supervisor	

1.由于默认kong启动是以daemon方式启动.所以修改/etc/kong/kong.conf配置文件

```
#将下列行修改为off,且取消注释
nginx_daemon = off
```

2.编辑kong的supervisor配置文件

```
[program:kong]
command=/usr/local/bin/kong start -c /etc/kong/kong.conf
numprocs=1
priority=0
user=work
stdout_logfile = /data/logs/kong/kong_supervisor.log
redirect_stderr = true
autostart=true
autorestart=true
```

> 注意由于kong启动的时候会连接后端的cassandra数据库,所以需要先启动cassandra,再启动kong.这就是为什么supervisor里要加入priority参数.优先级越小,启动顺序越优先.停止顺序越靠后. 	

**但是经过我的验证,发现priority参数没什么鸟用.当我start all,stop all时.永远是cassandra进程首先启动和关闭,无论priority优先级是大还是小.而不是supervisor官网介绍的那样效果**

启动没问题:

```
[work@kong-node2 ~]$ supervisorctl status
cassandra                        RUNNING   pid 13531, uptime 0:10:09
kong                             RUNNING   pid 14460, uptime 0:00:35
kong-dashboard                   RUNNING   pid 14496, uptime 0:00:17
```



但是发现supervisor管理kong进程有很严重的问题.

因为kong启动后包括2个进程:kong和nginx

```
work@kong-node1 conf.d]$ ps aux | grep kong
work      3198  0.2  0.0 130104  2700 ?        S    17:12   0:00 perl /usr/local/openresty/bin/resty /usr/local/bin/kong start -c /etc/kong/kong.conf
work      3215  6.3  0.1 259600 11800 ?        S    17:12   0:00 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /data/kong -c nginx.conf
```

.这个时候如果用supervisorctl restart kong进程会出现无法启动的情况.这是因为supervisor kill掉了kong进程.但是没有kill Ningx进程.所以重新启动kong的时候,由于nginx进程仍然存在,故无法启动.例如:

```
[work@kong-node1 conf.d]$ ps aux | grep kong
work      2917  0.0  0.1 259600 11820 ?        S    17:05   0:00 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /data/kong -c nginx.conf
work      3191  0.0  0.0 112704   976 pts/0    S+   17:11   0:00 grep --color=auto kong
[work@kong-node1 conf.d]$ kill 2917
[work@kong-node1 conf.d]$ ps aux | grep kong
work      3193  0.0  0.0 112704   976 pts/0    S+   17:11   0:00 grep --color=auto kong
[work@kong-node1 conf.d]$ supervisorctl start kong
kong: started

#kong启动后包含2个进程
[work@kong-node1 conf.d]$ ps aux | grep kong
work      3198  0.2  0.0 130104  2700 ?        S    17:12   0:00 perl /usr/local/openresty/bin/resty /usr/local/bin/kong start -c /etc/kong/kong.conf
work      3215  6.3  0.1 259600 11800 ?        S    17:12   0:00 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /data/kong -c nginx.conf
work      3228  0.0  0.0 112704   976 pts/0    R+   17:12   0:00 grep --color=auto kong

#supervisor关闭了Kong进程后,无法启动.
[work@kong-node1 conf.d]$ supervisorctl restart kong
kong: stopped
kong: ERROR (spawn error)

#因为虽然kong进程杀死了.但是nginx进程还在.所以kong的8000端口仍然被占用
[work@kong-node1 conf.d]$ ps aux | grep kong
work      3215  0.1  0.1 259600 11800 ?        S    17:12   0:00 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /data/kong -c nginx.conf
work      3307  0.0  0.0 112704   976 pts/0    S+   17:15   0:00 grep --color=auto kong

#查看启动日志,提示kong进程已经启动了.
Error: Kong is already running in /data/kong

  Run with --v (verbose) or --vv (debug) for more details
```

**暂时就不用supervisor管理了,采用命令行直接启动方式**

---

**命令行启动kong.只有一个Nginx进程.没有Perl进程.不知道何故**

```
[work@kong-node2 conf.d]$kong start -c /etc/kong/kong.conf
Kong started
[work@kong-node2 conf.d]$ps aux | grep kong
work     15558  0.0  0.0 259600  6540 ?        Ss   17:29   0:00 nginx: master process /usr/local/openresty/nginx/sbin/nginx -p /data/kong -c nginx.conf
```

