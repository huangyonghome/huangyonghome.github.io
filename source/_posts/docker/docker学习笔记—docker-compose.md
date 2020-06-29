---
title: docker学习笔记---docker-compose
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker学习笔记——docker-compose



docker compose 定义并且运行多个docker容器.使用YAML风格文件定义一个compose文件.利用compose文件创建和启动所有服务.

使用docker compose基本只需要3个步骤

- 在Dockerfile文件定义app环境
- 在docker-compose.yml文件中定义组成app的各个服务
- run docker-compose up 和compose 启动和运行app



下面文档均可以在docker-compose官方找到详细资料:[docker-compose](<https://docs.docker.com/compose/>)

### docker-compose安装

<!--more-->

docker-compose的安装非常简单.下面是Linux上的安装方法.其他平台请自行参考官网

1.下载最近的1.24版本的二进制文件

```
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```

2.给予执行权限.加入环境变量

```
sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

3.安装完成.查看是否安装成功

```
$ docker-compose --version
docker-compose version 1.24.0, build 1110ad01
```

---

## compose例子

在官网上,或者去github上下载一个例子.这里我参考<docker 深入浅出>这本书的例子

```
mkdir /data/counter-app
cd /data/counter-app
git clone https://github.com/nigelpoulton/counter-app.git
```

### docker-compose文件

```
[root@localhost counter-app]$cat docker-compose.yml

version: " 3.5"
services:
   web-fe:
      build:
         command: python app.py
         ports:
            - target: 5000
              published: 5000
         
         networks:
            - counter-net
         
         volumes:
            - type: volume
              source: counter-vol
              target: /code
    
   redis:
      image: "redis:alpine"
      networks:
         counter-net
   
networks:
        counter-net:
   
volumes:
       counter-vol:
```

**compose文件结构**

包含4个一级key: version.services.network.volumes

* version: 必须指定,定义了compose文件格式版本.这里是3.5最新版
* services: 用于定义不同的应用服务.这个例子中定义了2个服务.一个是web-fe的web前端.一个是redis的内存数据库.docker compose会将每个服务部署在各自的容器中
* networks用于创建新的网络.默认情况下会创建bridge网络
* volume用于创建新的卷



上面的docker compose文件定义了2个服务.在web-fe的服务定义中.包含如下指令:

* build:  指定docker基于当前目录下的Dockerfile文件构建一个新镜像
* command: 指定在容器中执行app.py脚本作为主程序 (这个指令可以忽略,因为dockerfile镜像中已经配置了CMD指令)
* ports: 将容器(target)的5000端口映射到宿主机(published)5000端口
* networks: docker将此容器连接到指定的网络上
* volumes: 指定docker将宿主机counter-vol卷(source)挂载到容器内的/code(target)上.counter-vol卷是已经存在的,或者是在文件下方的volumes一级key中定义的

redis服务比较简单,就不再赘述..

---

## 部署docker-compose

简要介绍counter-app目录内的几个文件

```
[root@localhost counter-app]$ll
总用量 20
-rw-r--r-- 1 root root 599 6月  18 17:34 app.py    #应用程序代码
-rw-r--r-- 1 root root 475 6月  17 18:46 docker-compose.ymal  #compose文件,定义了如何部署容器
-rw-r--r-- 1 root root 109 6月  18 17:34 Dockerfile  #构建web-fe服务镜像的dockerfile
-rw-r--r-- 1 root root 128 6月  18 17:34 README.md   
-rw-r--r-- 1 root root  11 6月  18 17:34 requirements.txt #列出app.py代码文件中python的依赖包
```

#### 启动docker-compose

在当前目录下执行下列路径

```
docker-compose up -d #后台启动
```

默认情况下```docker-compose```命令会寻找当前目录下名为docker-compose.yml或者docker-compose.yaml的Compose文件.如果Compose文件是其他文件名.则需要-f参数来指定具体文件名:

```
docker-compose -f compose_file up -d 
```

如果找不到文件则会报错:

```
ERROR:
        Can't find a suitable configuration file in this directory or any
        parent. Are you in the right directory?

        Supported filenames: docker-compose.yml, docker-compose.yaml
```

部署过程中创建或者拉取了3个镜像: counterapp_web-fe,python,redis

部署完成后,启动了如下2个容器:

```
[root@localhost counter-app]$docker ps
CONTAINER ID        IMAGE                             COMMAND                  CREATED             STATUS              PORTS                    NAMES
474301996ccc        redis:alpine                      "docker-entrypoint.s…"   21 hours ago        Up 21 hours         6379/tcp                 counter-app_redis_1
c7a1e28b5e28        counter-app_web-fe                "python app.py"          21 hours ago        Up 21 hours         0.0.0.0:5000->5000/tcp   counter-app_web-fe_1
```

每个容器都以项目名为前缀(所在目录名称).此外,还用一个数字为后缀用于表示容器序列(因为docker-compose允许扩容和缩减服务器数量)

同时,docker-compose还创建了counter-app_counter-net网络:

```
[root@localhost counter-app]$docker network ls
NETWORK ID          NAME                      DRIVER              SCOPE
6d40a81d76e7        bridge                    bridge              local
ef71284e9acc        counter-app_counter-net   bridge              local
```

应用部署成功后,可以查看容器的运行效果.每次访问,计数器就+1

```
[root@localhost counter-app]$curl http://localhost:5000
What's up Docker Deep Divers! You've visited me 1 times.
[root@localhost counter-app]$curl http://localhost:5000
What's up Docker Deep Divers! You've visited me 2 times.
[root@localhost counter-app]$curl http://localhost:5000
What's up Docker Deep Divers! You've visited me 3 times.
```

---

## docker Compose管理

上面讲到如何部署一个compose应用..接下来讲解一下compose的管理命令.需要注意的是所有的docker-compose命令都需要在相关目录下执行.不然仍然会提示找不到docker-compose.yml(yaml)文件

如果是停止应用.只需将up换成down即可.

```
[root@localhost counter-app]$docker-compose down
Stopping counter-app_redis_1  ... done
Stopping counter-app_web-fe_1 ... done
Removing counter-app_redis_1  ... done
Removing counter-app_web-fe_1 ... done
Removing network counter-app_counter-net
```

停止compose经历了如下的过程:

* 停止所有容器
* 移除容器
* 移除docker网络

此时,无论是执行```docker ps ```还是```docker ps -a```都看不到容器



#### 查看compose各个服务容器的运行的进程

```
[root@localhost counter-app]$docker-compose top
counter-app_redis_1
UID    PID    PPID    C   STIME   TTY     TIME         CMD
---------------------------------------------------------------
100   32558   32542   0   13:21   ?     00:00:00   redis-server

counter-app_web-fe_1
UID     PID    PPID    C   STIME   TTY     TIME                    CMD
--------------------------------------------------------------------------------------
root   32582   32564   6   13:21   ?     00:00:00   python app.py
root   32703   32582   4   13:21   ?     00:00:00   /usr/local/bin/python /code/app.py
[root@localhost counter-app]$
```

> PID是docker宿主机的进程ID



#### 停止应用容器.但是并不删除资源

执行完```docker-compose stop```命令后,容器还存在

```
[root@localhost counter-app]$docker-compose stop
Stopping counter-app_web-fe_1 ... done
Stopping counter-app_redis_1  ... done

[root@localhost counter-app]$docker-compose ps
        Name                      Command               State    Ports
----------------------------------------------------------------------
counter-app_redis_1    docker-entrypoint.sh redis ...   Exit 0
counter-app_web-fe_1   python app.py                    Exit 0
[root@localhost counter-app]$
```



#### 删除,重启已停止的compose应用容器

```
docker-compose rm #删除.删除应用相关的容器,但是不会删除卷和镜像和网络.
docker-compose restart #重启
```

#### 

#### 拉取服务镜像

```docker-compose pull server_name ```这个命令会先拉取服务镜像到本地.例如在本文的docker compose例子中有2个服务:web-fe和redis.如果执行下列命令,会仅仅拉取redis镜像到本地

```
docker-compose pull redis
```



#### 其他命令

日常docker管理容器的命令都可以使用```docker-compose```替代.例如:

```
docker-compose logs service_name #查看服务容器日志
docker-compose exec service_name #开启终端登陆容器
docker-compose kill -s SIGINT    #杀死docker-compose服务容器
docker-compose ps                #列出容器
```

---

### docker-compose配置文件指令解析

以下配置文件以版本3.x为例.官网参考:<https://docs.docker.com/compose/compose-file/>

下面是个包含完整指令的样例:

```
version: "3.7"
services:

  redis:
    image: redis:alpine
    ports:
      - "6379"
    networks:
      - frontend
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  db:
    image: postgres:9.4
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]

  vote:
    image: dockersamples/examplevotingapp_vote:before
    ports:
      - "5000:80"
    networks:
      - frontend
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure

  result:
    image: dockersamples/examplevotingapp_result:before
    ports:
      - "5001:80"
    networks:
      - backend
    depends_on:
      - db
    deploy:
      replicas: 1
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 1
      labels: [APP=VOTING]
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    stop_grace_period: 1m30s
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]

networks:
  frontend:
  backend:

volumes:
  db-data:
```



## Service块级别的配置文件指令

#### build 

build可以指定一个目录或者在build下还可以指定context上下文环境和docker-file文件名

```
version: "3.7"
services:
  webapp:
    build: ./dir
    
或者
webapp:
    build:
      context: ./dir
      dockerfile: Dockerfile-alternate #如果指定dockerfile,则必须要指定一个build路径,也就是context
      args:
        buildno: 1
```

如果同时指定了Image关键字,那么会构建一个指定的镜像名:tag

```
build: ./dir
image: webapp:tag  #构建webapp:tag的镜像名
```

> build选项在swarm中部署stack时是无效的,因为docker stack命令只接受已经build好的镜像

#### CONTEXT

定义上下文目录.如果是一个相对目录,那么是相对Compose file文件的目录.

#### ARGS

在build过程中可以允许使用ARGS变量传递给dockerfile.具体用法参考官网

#### COMMAND

重写dockerfile或者镜像中的默认命令.和dockerfile一样可以是shell方式也可以是exec方式执行

```
command: bundle exec thin -p 3000
command: ["bundle", "exec", "thin", "-p", "3000"]
```

#### configs

授予每个service的配置文件访问.具体用法参考官网

#### container_name

指定一个容器名,而不是使用默认名字

```
container_name: my-web-container
```

> 需要注意的是由于容器名必须唯一,所以当扩展多个容器副本时,指定一个具体的容器名会报错,所以这个指令在swarm模式下部署stack时会被忽略

#### depends_on

用于在多个services之间指定依赖性.service dependencies会导致以下行为

* ```docker-compose up``` 启动时会参考depndency顺序.在下面这个例子中.db和redis服务会先于web服务启动
* ```docker-compose up SERVICE``` 会自动启动该SERVICE的依赖服务.在下面例子中```docker-compose up web```命令会自动创建和启动db和redis
* ```docker-compose stop```会参考依赖顺序而停止服务.在下面例子中,web服务会先于db和redis服务停止

```
version: "3.7"
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```

> 使用depens_on需要注意以下几点:
>
> 1.denpds_on只会在web依赖的服务启动后就启动web服务,而不是等待db和redis服务启动并且处于ready状态才启动web.这有可能会带来一些问题,比如mysql启动较慢,数据库还没准备好等.如果你需要确定后端的db,redis数据库启动成功,并且可以连接时才启动web服务,可以参考https://docs.docker.com/compose/startup-order/
>
> 2.version3版本不再支持depends_on下的condition指令
>
> 3.version3版本的depends_on选项在swarm模式下部署stack时会被忽略

depends_on选项的控制启动顺序参考:

编写一个shell脚本循环判断后端的数据库是否ready.如果ready则执行CMD命令.然后在command指令中指定脚本的后端db数据库服务名,以及CMD命令参数

```
#!/bin/sh
# wait-for-postgres.sh
#循环测试db服务($1参数)的状态.一旦可以连接了,执行cmd命令(python app.py)
set -e

host="$1"
shift
cmd="$@"

until PGPASSWORD=$POSTGRES_PASSWORD psql -h "$host" -U "postgres" -c '\q'; do
  >&2 echo "Postgres is unavailable - sleeping"
  sleep 1
done

>&2 echo "Postgres is up - executing command"
exec $cmd
```

```
command: ["./wait-for-postgres.sh", "db", "python", "app.py"]
```

#### deploy指令

该指令用于配置服务相关的配置和部署方式.这个指令只在version3版本支持,而且只在swarm模式下才生效.单机```docker-compose up```方式执行会被忽略

```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      replicas: 6
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
```

下面是有关deploy的几个子指令介绍

* ENDPOINT_MODE

> version 3.3 only

```endpoint_mode: VIP```  Docker为service分配一个虚拟IP.作为用户的前端入口.docker路由用户请求到所有可用的worker节点..这也是默认模式

```endpoint_mode: dnsrr``` DNS轮询服务,Docker发起一个service name的DNS查询,并且返回一个包含多个IP地址的列表.客户端通过轮询方式链接其中一个IP地址.

* LABLES

为service指定一个标签.只对service生效,无法为service的具体某个容器生效

* MODE

mode定义了在swarm节点上的副本部署模式,有global和replicated两种模式,默认是replicated

```
version: "3.7"
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    deploy:
      mode: global
```

关于两种模式的区别参考:<https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/#replicated-and-global-services>

* REPLACEMENT

定义了constrants(约束条件)和preferences.的参数.

下面是constrants指令的用法:

constrants指令可以限制某个task在哪些swarm节点上运行.多个constrants指令是逻辑AND的关系来匹配满足条件的nodes.constrants可以匹配swarm节点或者Docker引擎标签:



| node attribute  | matches                  | example                                       |
| :-------------- | :----------------------- | :-------------------------------------------- |
| `node.id`       | Node ID                  | `node.id==2ivku8v2gvtg4`                      |
| `node.hostname` | Node hostname            | `node.hostname!=node-2`                       |
| `node.role`     | Node role                | `node.role==manager`                          |
| `node.labels`   | user defined node labels | `node.labels.security==high`                  |
| `engine.labels` | Docker Engine's labels   | `engine.labels.operatingsystem==ubuntu 14.04` |

例如下面这个例子中限制redis service的task运行在lable标签等于queue的swarm节点

```
$ docker service create \
  --name redis_2 \
  --constraint 'node.labels.type == queue' \
  redis:3.0.6
```

回到刚才REPLACEMENT的例子,下面的例子中表示db service只运行在swarm manager节点,而且docker node节点的操作系统是Ubuntu 14.04

```
version: "3.7"
services:
  db:
    image: postgres
    deploy:
      placement:
        constraints:
          - node.role == manager
          - engine.labels.operatingsystem == ubuntu 14.04
        preferences:
          - spread: node.labels.zone
```

* REPLICAS

如果service 是replicated模式(默认模式),定义容器的启动数量.在下面的例子中启动6个worker容器

```
version: "3.7"
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 6
```

* RESOURCES

配置容器限定的使用资源

在下面这个例子中.redis service被限制只允许使用不超过50M内存,以及0.5的CPU处理器时间(单个CPU内核的50%).并且有20M内存和0.25的CPU处理器时间预留(也就是永远为redis service保留)

```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
        reservations:
          cpus: '0.25'
          memory: 20M
```

**Out Of Memory Exceptions[OOME]**

如果service 或者容器使用了超过限制的资源.容器或者docker引擎就会出现OOME错误.docker进程可能会被内核OOM killer给kill掉.关于如何规避这种问题,请参考[understand the risks of running out of memory](<https://docs.docker.com/config/containers/resource_constraints/>)

* RESTART_POLICY

配置当容器停止时如何重新启动容器的策略.有以下几种子指令

1.```condition``` 重启容器的约束条件.有: ```none```,```on-failure```,和```any```(default:any)

2.```delay```: 尝试重启容器的时间间隔.默认是0

3.```max_attempts```:如果容器重启失败,重启最大尝试次数,默认是一直尝试

4.```window```:重启后等待多久认定重启成功.默认是immediately

```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```



#### ROLLBACK_CONFIG

更新失败的回滚指令.

#### UPDATE_CONFIG

配置service如何进行滚动更新.

#### DNS

自定义DNS地址,可以是单个值或者一个列表

```
dns: 8.8.8.8

dns:
  - 8.8.8.8
  - 9.9.9.9
```

#### entrypoint

重写dockerfile或者镜像中的entrypoint指令

```
entrypoint: /code/entrypoint.sh

#也可以是一个列表格式:

entrypoint:
    - php
    - -d
    - zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20100525/xdebug.so
    - -d
    - memory_limit=-1
    - vendor/bin/phpunit
```

#### env_file

添加环境变量文件,可以是单个值,也可以是个列表.该文件最好是在当前docker-compose文件目录或者子目录下.

如果是指定多个变量文件,而且有重复的变量且赋值不同,那么以最后一个变量文件的变量为准

```
services:
  some-service:
    env_file:
      - a.env
      - b.env
```

#### environment

添加环境变量.可以使用列表格式,或者字典格式

```
environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SHOW=true
  - SESSION_SECRET
```

#### extra_hosts

添加hostname和IP地址的绑定映射到hosts文件

```
extra_hosts:
 - "somehost:162.242.195.82"
 - "otherhost:50.31.209.229"
 
 #会在容器内的/etc/hosts文件生成如何内容
162.242.195.82  somehost
50.31.209.229   otherhost
```

#### healthcheck

检查service的各个容器是否处于"healthy"状态.例如下面的例子

```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

```test```指令必须为一个字符串或者一个列表.如果是个上例子中的列表格式.则第一个参数必须为```NONE```,```CMD```,或者```CMD-SHELL```.如果是字符串相当于指定了```CMD-SHELL```参数

下面2个写法和上文的例子效果一样

```
test: ["CMD-SHELL", "curl -f http://localhost || exit 1"]
```

```
test: curl -f https://localhost || exit 1
```

要关闭健康检查,可以使用```disable:true```.等同于```test:["NONE"]```

```
healthcheck:
  disable: true
```

#### image

指定容器的启动镜像.可以是指定的repository/tag 或者一个镜像ID.如果本地不存在该镜像,会尝试去pull镜像到本地.如果指定了```build```指定,会使用指定的命令来构建一个镜像

下面这几种写法均正确

```
image: redis
image: ubuntu:14.04
image: tutum/influxdb
image: example-registry.com:4000/postgresql
image: a4bc65fd
```

#### logging

service的log配置.下面的例子中指定了一个syslog服务器的地址

```
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```

```driver```为容器指定logging的驱动,一共以下3种驱动方式,默认是json-file

```
driver: "json-file"
driver: "syslog"
driver: "none"
```

也可以限定json-file驱动的日志转出.例如下列指定了最大的日志文件大小和日志保留份数

```
options:
  max-size: "200k"
  max-file: "10"
```

#### network_mode

指定网络模式,有以下几种网络模式

```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```

#### networks

services加入的网络.这些网络名在顶级```network```指令中有指定

```
services:
  some-service:
    networks:
     - some-network
     - other-network
```

#### IPV4_ADDRESS,IPV6_ADDRESS

为容器指定一个静态的IP地址.但是对应的Network顶级指令中必须指定一个ipam块,定义该网络的IP子网范围.例如

app_net网络的ipam快中指定了172.16.238.0/24的子网.然后为app service指定一个静态IP

```
version: "3.7"

services:
  app:
    image: nginx:alpine
    networks:
      app_net:
        ipv4_address: 172.16.238.10
        ipv6_address: 2001:3984:3989::10

networks:
  app_net:
    ipam:
      driver: default
      config:
        - subnet: "172.16.238.0/24"
        - subnet: "2001:3984:3989::/64"
```



#### ports

暴露端口.下面是几种短格式写法.推荐将端口用双引号括起来

```
ports:
 - "3000"
 - "3000-3005"
 - "8000:8000"
 - "9090-9091:8080-8081"
 - "49100:22"
 - "127.0.0.1:8001:8001"
 - "127.0.0.1:5000-5010:5000-5010"
 - "6060:6060/udp"
```

> 此外还有完整格式的写法.



#### restart

默认的restart策略是```no```有以下四种重启策略



```
restart: "no"
restart: always
restart: on-failure
restart: unless-stopped
```

### sysctls

配置容器的内核参数.可以是数组或者字典类型

```
sysctls:
  net.core.somaxconn: 1024
  net.ipv4.tcp_syncookies: 0
  
sysctls:
  - net.core.somaxconn=1024
  - net.ipv4.tcp_syncookies=0
```

### ulimits

配置ulimits

```
ulimits:
  nproc: 65535
  nofile:
    soft: 20000
    hard: 40000
```

### volumes

挂载一个路径或者一个卷.

如果是在service层级挂载宿主机上的路径到容器,那么不需要在顶级指令中定义```volumes```key.但是如果是挂载一个卷到多个service,可以在顶级指令中定义个卷名.然后使用这个卷名去挂载

下面这个例子在顶级```volumes```指令中定义了2个卷名:mydata和dbdata. mydata被web service挂载.dbdata被db service挂载.下面2个挂载格式都可以.

```
version: "3.7"
services:
  web:
    image: nginx:alpine
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

  db:
    image: postgres:latest
    volumes:
      - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock"
      - "dbdata:/var/lib/postgresql/data"

volumes:
  mydata:
  dbdata:
```

简短格式写法

```
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql

  # Specify an absolute path mapping
  - /opt/data:/var/lib/mysql

  # Path on the host, relative to the Compose file
  - ./cache:/tmp/cache

  # User-relative path
  - ~/configs:/etc/configs/:ro

  # Named volume
  - datavolume:/var/lib/mysql
```



完整格式写法

```
version: "3.7"
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

networks:
  webnet:

volumes:
  mydata:
```

---

## Volume块级别配置文件指令

大部分volume的用法在上面都已经解释过了,Volume的顶级指令配置不多.

#### driver

指定volume卷的驱动,docker引擎默认指定的驱动是```local```.

---

## Network块级别配置文件指令

详细的docker network特性以及所有network驱动请见:[Network guide](<https://docs.docker.com/compose/networking/>)

默认情况下Compose启动单一网络,一个services的每个容器加入到默认的网络,并且该网络下的所有容器之间都能互相访问

假如下面的compose文件在```myapp```目录下.

```
version: "3"
services:
  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres
    ports:
      - "8001:5432"
```

当执行```docker-compose up```命令启动时,会执行以下几个步骤

1.创建一个```myapp_default```的网络

2.使用web的配置文件启动一个容器,加入到```myapp_default```网络中

3.使用db的配置文件启动一个容器.加入到```myapp_default```的网络中

所有容器成功启动后,每个容器都能访问对方的```hostname```和对方的IP地址.

另外,需要理解```HOST_PORT```和```COMTAINER_PORT```的区别.在上面这个例子中,db的```host_port```是8001,容器的端口是5432,services之间的容器都是通过```CONTAINER_PORT```也就是容器的IP进行通信的.例如访问数据库地址应该是:```postgres://db:5432```



#### driver

指定网络驱动.在单主机下Docker引擎默认使用```bridge```模式.在swarm集群环境中,默认使用```overlay```模式

```
driver: overlay
```

---

## configs和secrets块级别配置文件指令

还不是很懂这个怎么用的,以后有空再研究