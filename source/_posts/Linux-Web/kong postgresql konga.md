---
title: kong+postgresql+konga集群环境部署
date: 2020-06-26 11:59:58
tags:  kong
categories: [Linux-Web, kong]
comments: true
copyright: true
---



## kong+postgresql+konga集群环境部署

### kong简介
Kong是Mashape开源的一款API网关，起初是用来管理 Mashape 公司15000个微服务的，后来在2015年开源,现在已经在很多创业公司、大型企业和政府机构中广泛使用。基于nginx,Lua和Cassandra或PostgreSQL，支持分布式操作，有很强的可移植性和可扩展性。可以在任何一种基础设施上运行,作为应用和API之间的中间层，加上众多功能强大的插件，可以实现认证授权、访问控制等功能。并且提供易于使用的RESTful API来操作和配置系统。

有关kong的详细介绍请参考官网.

--

### postgreSQL简介
[PostgreSQL](https://baike.baidu.com/item/PostgreSQL/530240) 是一个免费的对象-关系数据库服务器(数据库管理系统)，它在灵活的 BSD-风格许可证下发行。它提供了相对其他开放源代码数据库系统(比如 MySQL 和 Firebird)，和专有系统(比如 Oracle、Sybase、IBM 的 DB2 和 Microsoft SQL Server)之外的另一种选择。

--

### 集群架构

* **kong cluster**

kong 集群并不意味着客户端请求将会负载均衡到kong集群中的每个节点上，kong集群并不是开箱即用，仍然需要在kong集群多节点上层搭建负载均衡，以便分发请求。 一个kong集群只是意味着集群内的节点，都共享同样的配置。

有关Kong cluster集群的详细介绍请参考官网:[Kong cluser document](https://docs.konghq.com/0.14.x/clustering/)

为了提高冗余性和健壮性.我们对kong的每个环节都进行了冗余设计.一个基本的kong集群架构大概如下图所示:

![](https://img1.jesse.top/kong-flow.png)

<!--more-->

--

### 部署步骤

**环境**: 

阿里云ECS Centos7.4操作系统  
Kong: 1.0最新版
postgresql 9.6

konga 最新版

**集群架构说明:**

dwd-kong-node1节点服务器部署postgresql master主库

dwd-kong-node2节点服务器部署postgresql slave从库

在两个Kong节点服务器上都部署konga,但是konga指向postgresql master主库

--

#### 安装postgresql

安装方式官网参考: [Installing Postgresql](<https://www.postgresql.org/>)



**安装步骤**

1.安装Postgre9.6版本的Yum源

```
yum install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm
```

2.安装postgresql

```
sudo yum install postgresql96 postgresql96-server -y 
```

3.切换到root用户下,初始化数据库

```
sudo /usr/pgsql-9.6/bin/postgresql96-setup initdb
```

4.修改数据库的配置文件,监听所有接口

postgresql的配置文件默认路径在:/var/lib/pgsql/9.6/data/postgresql.conf

修改下列配置:

```
listen_addresses = '*' #监听所有接口.由于我服务器只有内网地址,所有可以侦听在所有接口,(如果有公网地址,最好不要只样做)
log_directory = '/data/logs/postgre' #指定日志文件的父目录

```

5.修改数据库远程访问配置文件.开启远程访问

```
vim /var/lib/pgsql/9.6/data/postgresql.conf/pg_hba.conf

修改下列两项:
#修改为md5认证,下列10.0.0.0/8是内网地址段
host    all             all             127.0.0.1/32            md5
host    all             all             10.0.0.0/8           md5
```

6.创建日志目录,并且赋权给postgres用户(安装postgresql后默认会创建这个用户)

```
[root@dwd-kong-node1 data]# mkdir /data/logs/postgre
[root@dwd-kong-node1 data]# chown -R postgre.postgre /data/logs/postgre
```

7.启动postgresql数据库

```
systemctl enable postgresql-9.6
systemctl start postgresql-9.6
```

---

### 创建数据库用户密码

1.切换到postgres用户,输入psql登陆数据库

```
[root@dwd-kong-node1 data]# su - postgres
Last login: Mon Apr  8 09:27:12 CST 2019 on pts/0
-bash-4.2$
-bash-4.2$ psql
psql (11.2, server 9.6.12)
Type "help" for help.

postgres=#
```

2.创建kong用户和kong数据库

```
 CREATE USER kong; 
 CREATE DATABASE kong OWNER kong;
```

3.创建konga的数据库

```
 CREATE DATABASE konga OWNER kong;
```

4.为kong用户创建一个密码,密码也是kong

```
alter user kong with password 'kong';
```

5.输入\l,可以看到当前的数据库

```
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 kong      | kong     | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 konga     | kong     | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(5 rows)
```

6.查看当前数据库有哪些用户

```
postgres=# select * from pg_roles;
      rolname      | rolsuper | rolinherit | rolcreaterole | rolcreatedb | rolcanlogin | rolreplication | rolconnlimit | rolpassword | rolvaliduntil | rolbypassrls | rolconfig |  oid
-------------------+----------+------------+---------------+-------------+-------------+----------------+--------------+-------------+---------------+--------------+-----------+-------
 postgres          | t        | t          | t             | t           | t           | t              |           -1 | ********    |               | t            |           |    10
 pg_signal_backend | f        | t          | f             | f           | f           | f              |           -1 | ********    |               | f            |           |  4200
 kong              | f        | t          | f             | f           | t           | f              |           -1 | ********    |               | f            |           | 16384
(3 rows)
```

---


#### 安装Kong

安装方法可以参考官网:[Install Kong](https://docs.konghq.com/install/centos/?_ga=2.110797315.728319704.1539597667-917309945.1539077269#packages)

1.下载,安装rpm安装包

```
wget -O kong-community-edition-1.0.0.el7.noarch.rpm  https://kong.bintray.com/kong-community-edition-rpm/centos/7/:kong-community-edition-1.0.0.el7.noarch.rpm

sudo yum install epel-release
sudo yum install kong-community-edition-1.0.0.el7.noarch.rpm
```

2.dwd-kong-node1修改kong配置文件

```
cd /etc/kong
sudo cp kong.conf.default kong.conf

[root@dwd-kong-node1 data]#  sed -r '/^[[:space:]]+#*/d' /etc/kong/kong.conf | sed '/^#/d' | sed '/^$/d'

prefix = /data/logs/kong/       # Working directory. Equivalent to Nginx's
proxy_access_log = access.log       # Path for proxy port request access
proxy_error_log = error.log         # Path for proxy port request error
admin_listen = 0.0.0.0:8001     # Address and port on which Kong will expose
database = postgres             # Determines which of PostgreSQL or Cassandra
pg_host = 127.0.0.1             # The PostgreSQL host to connect to.
pg_port = 5432                  # The port to connect to.
pg_user = kong                  # The username to authenticate if required.
pg_password = kong                 # The password to authenticate if required.
pg_database = kong              # The database name to connect to.
cluster_listen = 0.0.0.0:7946   # Address and port used to communicate with
cluster_listen_rpc = 127.0.0.1:7373  # Address and port used to communicate
```

3.dwd-kong-node2修改kong配置文件

```
[root@dwd-kong-node2 system]# clear
[root@dwd-kong-node2 system]# sed -r '/^[[:space:]]+#*/d' /etc/kong/kong.conf | sed '/^#/d' | sed '/^$/d'
prefix = /data/logs/kong/       # Working directory. Equivalent to Nginx's
proxy_access_log = access.log       # Path for proxy port request access
proxy_error_log = error.log         # Path for proxy port request error
database = postgres             # Determines which of PostgreSQL or Cassandra
pg_host = 10.111.30.174             # The PostgreSQL host to connect to.
pg_port = 5432                  # The port to connect to.
pg_user = kong                  # The username to authenticate if required.
pg_password = kong                 # The password to authenticate if required.
pg_database = kong              # The database name to connect to.
pg_ssl = off                    # Toggles client-server TLS connections
pg_ssl_verify = off             # Toggles server certificate verification if
cluster_listen = 0.0.0.0:7946   # Address and port used to communicate with
cluster_listen_rpc = 127.0.0.1:7373  # Address and port used to communicate
```

> 唯一区别就是这里的数据库指向dwd-kong-node1上的postgresql(IP:10.111.30.174),而非本机.



3.创建kong目录

```
mkdir /data/logs/kong/
```

4.在dwd-kong-node1上准备数据库

```
kong migrations bootstrap -c /etc/kong/kong.conf
```
> 由于dwd-kong-node2上指向了node1的数据库,所以在node2上不需要执行这个命令



5.启动kong

```
kong start -c /etc/kong/kong.conf
```

查看端口.可以看到postgresql和kong的侦听端口都已经成功启动

```
[root@dwd-kong-node1 data]# netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:5432            0.0.0.0:*               LISTEN      26580/postmaster
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      24746/nginx: master
tcp        0      0 0.0.0.0:5822            0.0.0.0:*               LISTEN      29688/sshd
tcp        0      0 0.0.0.0:8000            0.0.0.0:*               LISTEN      24746/nginx: master
tcp        0      0 0.0.0.0:8001            0.0.0.0:*               LISTEN      24746/nginx: master
tcp6       0      0 :::5432                 :::*                    LISTEN      26580/postmaster
udp        0      0 0.0.0.0:68              0.0.0.0:*                           748/dhclient
udp        0      0 10.111.30.174:123       0.0.0.0:*                           6290/ntpd
udp        0      0 127.0.0.1:123           0.0.0.0:*                           6290/ntpd
udp        0      0 0.0.0.0:123             0.0.0.0:*                           6290/ntpd
udp        0      0 0.0.0.0:34019           0.0.0.0:*                           748/dhclient
udp6       0      0 :::31421                :::*                                748/dhclient
udp6       0      0 :::123                  :::*                                6290/ntpd
```

---

### 搭建konga

konga是管理kong各个组件(serveice,route,plugin,upstream,consumer)的可视化UI管理工具.在增删改查各个组件的配置时非常方便.

个人觉得UI界面比kong-dashboard要漂亮

konga的github参考:[konga](https://github.com/pantsel/konga)

---

1.确保需要有node.js环境.如果没有npm工具,必须先安装nodejs

```
[work@DWD-BETA kong]$ npm -v
6.4.1
[work@DWD-BETA kong]$ node -v
v10.15.3
```

2.安装bower,gulp包.安装git软件

```
npm install bower
npm install gulp
yum install git
```

3.work用户下安装konga

```
$ git clone https://github.com/pantsel/konga.git
$ cd konga
$ npm i
```
4.编辑.env环境文件(dwd-kong-node2的文件内容中将下列的localhost修改为node1服务器的IP:10.111.30.174)

```
work@dwd-kong-node1 konga]$ cat .env_example
PORT=1337
NODE_ENV=production
KONGA_HOOK_TIMEOUT=120000
DB_ADAPTER=postgres
DB_URI=postgresql://localhost:5432/konga
KONGA_LOG_LEVEL=warn
TOKEN_SECRET=some_secret_token
DB_USER=kong
DB_PASSWORD=kong
DB_DATABASE=konga
```

5.启动konga

```
work@dwd-kong-node1 konga]$ npm start
```
6.如果启动报错,则安装依赖

```
work@dwd-kong-node1 konga]$ npm run bower-deps
```
这个程序默认是前台启动

```
[work@dwd-kong-node1 konga]$ npm start

> kongadmin@0.14.3 start /home/work/konga
> node --harmony app.js

No DB Adapter defined. Using localDB...
debug: Hook:api_health_checks:process() called
debug: Hook:health_checks:process() called
debug: Hook:start-scheduled-snapshots:process() called
debug: Hook:upstream_health_checks:process() called
debug: Hook:user_events_hook:process() called
debug: User had models, so no seed needed
debug: Kongnode had models, so no seed needed
debug: Emailtransport seeds updated
debug: -------------------------------------------------------
debug: :: Mon Apr 08 2019 18:34:31 GMT+0800 (China Standard Time)
debug: Environment : development
debug: Port        : 1337
debug: -------------------------------------------------------

```



此时在浏览器输入:http://10.111.30.174:1337 就能访问konga了.

---

### systemctl管理kong和konga进程

* kong

在/usr/lib/systemd/system路径下编辑kong.service文件.内容如下:

```
work@dwd-kong-node1 system]$ pwd
/usr/lib/systemd/system
[work@dwd-kong-node1 system]$ cat kong.service
[Unit]
Description= kong service
After=syslog.target network.target postgresql-9.6.target


[Service]
User=work
Group=work
Type=forking
ExecStart=/usr/local/bin/kong start -c /etc/kong/kong.conf
ExecReload=/usr/local/bin/kong reload -c /etc/kong/kong.conf
ExecStop=/usr/local/bin/kong stop
Restart=always

[Install]
WantedBy=multi-user.target
```

* konga

在/usr/lib/systemd/system路径下编辑konga.service文件.内容如下:

```
[work@dwd-kong-node1 system]$ cat konga.service
[Unit]
Description= konga service
After=syslog.target network.target postgresql-9.6.target  kong.target


[Service]
User=work
Group=work
Type=forking
#需要指定工作目录,因为npm命令要在konga的目录下执行
WorkingDirectory=/home/work/konga
ExecStart=/usr/local/bin/npm start
ExecStop=kill $(netstat -tlnp |grep 1337|  awk '{print $NF}' | awk -F "/" '{print $1}')
Restart=always

[Install]
WantedBy=multi-user.target
```

* 启动进程,且设置开机启动

```
[work@dwd-kong-node2 ~]$ sudo systemctl enable kong
Created symlink from /etc/systemd/system/multi-user.target.wants/kong.service to /usr/lib/systemd/system/kong.service.
[work@dwd-kong-node2 ~]$ sudo systemctl enable konga.service
Created symlink from /etc/systemd/system/multi-user.target.wants/konga.service to /usr/lib/systemd/system/konga.service.
[work@dwd-kong-node2 ~]$ sudo systemctl start kong.service
[work@dwd-kong-node2 ~]$ sudo systemctl start konga.service

```

---

