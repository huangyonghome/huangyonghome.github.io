---
title: docker学习笔记---理解swarm集群
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



### 理解swarm集群

一个swarm是一组运行docker服务器的的集群,docker服务器可以是物理机也可以是虚拟机.

swarm manager可以使用多种策略来运行容器.比如"emptiest node"---部署容器到压力最小的服务器上,或者"global"---确保每台服务器都只允许一个容器实例.你可以在Compose文件中指示swarm manager去选择何种策略

swarm managers是swarm进群中唯一可以执行命令,或者授权其他服务器以"workers"身份加入swarm集群的服务器.

---

#### 初始化swarm,加入节点

试验环境:

1.10.0.0.50 ---swarm manager  
2.10.0.0.12 ---worker 节点

* 初始化swarm,并且指定通告的IP

<!--more-->

```
[root@localhost ~]$docker swarm init --advertise-addr 10.0.0.50
Swarm initialized: current node (zlvl9l94blu3rfcaaptdvo9u1) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-2i5fyjf2niw81tudcvpw33yuni277vz45lt6tyi5bvcnhvuwea-bj091dpe6e69ph9kt3lmsthgp 10.0.0.50:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

根据上面提示,在第二台服务器上以worker身份加入swarm集群

```
[root@php ~]$docker swarm join --token SWMTKN-1-2i5fyjf2niw81tudcvpw33yuni277vz45lt6tyi5bvcnhvuwea-bj091dpe6e69ph9kt3lmsthgp 10.0.0.50:2377
This node joined a swarm as a worker.
[root@php ~]$
```

执行docker node ls命令可以管理和查看swarm集群的所有节点

```
[root@localhost ~]$docker node ls
ID                            HOSTNAME                STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zlvl9l94blu3rfcaaptdvo9u1 *   localhost.localdomain   Ready               Active              Leader              18.09.2
ud5ztqzvvfwg3d3hwmts5y9ct     php                     Ready               Active                                  18.09.3
[root@localhost ~]$
```

执行docker swarm leave命令将某个节点退出swarm集群

```
[root@php ~]$docker swarm leave
Node left the swarm.
[root@php ~]$

此时这个节点在swarm集群中状态为down

[root@localhost ~]$docker node ls
ID                            HOSTNAME                STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
zlvl9l94blu3rfcaaptdvo9u1 *   localhost.localdomain   Ready               Active              Leader              18.09.2
ud5ztqzvvfwg3d3hwmts5y9ct     php                     Down                Active                                  18.09.3
[root@localhost ~]$

```
---

#### 在swarm集群部署app

现在可以把上一小节的docker compose部署在swarm集群上了.执行命令和上一小节一样.但是需要注意的是只能在swarm manager节点服务器上执行命令.

在第一台服务器上执行如下命令:(确保docker compose文件和镜像文件在这台服务器上)

```
root@localhost ~]$cd /data/compose
[root@localhost compose]$ls
docker-compose.yml
[root@localhost compose]$docker stack deploy -c docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web
[root@localhost compose]$
```

APP已经成功部署到swarm集群上,现在可以使用上一小节中的同样的命令来管理app集群,只不过这次services和容器已经部署到两台服务器上:

```
[root@localhost compose]$docker stack ps getstartedlab
ID                  NAME                      IMAGE                  NODE                    DESIRED STATE       CURRENT STATE              ERROR                              PORTS
4borwslue8k5        getstartedlab_web.1       friendlyhello:latest   localhost.localdomain   Running             Preparing 6 seconds ago
ixmkldh16otx        getstartedlab_web.2       friendlyhello:latest   php                     Ready               Preparing 22 seconds ago
i2yhimkst4iq         \_ getstartedlab_web.2   friendlyhello:latest   php                     Shutdown            Rejected 23 seconds ago    "No such image: friendlyhello:…"
gytqpcwnzvrm        getstartedlab_web.3       friendlyhello:latest   localhost.localdomain   Running             Preparing 6 seconds ago
j94tblo4qjwa        getstartedlab_web.4       friendlyhello:latest   php                     Ready               Preparing 22 seconds ago
b7r9xkf4glh6         \_ getstartedlab_web.4   friendlyhello:latest   php                     Shutdown            Rejected 22 seconds ago    "No such image: friendlyhello:…"
i8sv4c293ata        getstartedlab_web.5       friendlyhello:latest   localhost.localdomain   Running             Preparing 6 seconds ago
```

> 需要在另外一台服务器上pull同样的镜像,否则容器无法启动

* 在第二台服务器上下载我阿里云私有仓库的镜像
```
docker login --username=jessehuang408 registry.cn-hangzhou.aliyuncs.com
Password:

[root@php ~]$docker pull registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:frendlyhello-v1.0
frendlyhello-v1.0: Pulling from jesse_images/jesse_images
f7e2b70d04ae: Pull complete
1e9214730e83: Pull complete
5bd4ec081f7b: Pull complete
be26b369a1e7: Pull complete
236be9d80905: Pull complete
1bf8a3675b0b: Pull complete
5752f9477f0c: Pull complete
Digest: sha256:8e8b57ef6e22c8c04c1c80cfab9f336928cffabacaa4ae4e74ec57e54bcffdb2
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:frendlyhello-v1.0

[root@php ~]$docker images
REPOSITORY                                                    TAG                 IMAGE ID            CREATED             SIZE
registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images   frendlyhello-v1.0   f091d1bb803c        2 days ago          131MB
[root@php ~]$*

```

* 将镜像修改成和第一台服务器一样:frendlyhello:latest

命令:

```
docker tag 镜像ID REPOSITORY:TAG
```

```
[root@php ~]$docker tag f091d1bb803c frendlyhello:latest
[root@php ~]$docker images
REPOSITORY                                                    TAG                 IMAGE ID            CREATED             SIZE
frendlyhello                                                  latest              f091d1bb803c        2 days ago          131MB
registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images   frendlyhello-v1.0   f091d1bb803c        2 days ago          131MB
[root@php ~]$
```

* 将第一台服务器的docker-compose文件拷贝到同样的目录下

```
[root@php ~]$mkdir /data/compose
[root@php ~]$scp root@10.0.0.50:/data/compose/docker-compose.yml /data/compose/
```

* 回到第一台服务器上删除刚才创建的getstartedlab

```
[root@localhost compose]$docker stack rm getstartedlab
Removing service getstartedlab_web
Removing network getstartedlab_webnet

[root@localhost compose]$docker stack ps getstartedlab
nothing found in stack: getstartedlab
```

* 重新部署docker compose

```
[root@localhost compose]$docker stack deploy -c docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web
[root@localhost compose]$
```

* 成功部署

```
[root@localhost compose]$docker stack ps getstartedlab
ID                  NAME                  IMAGE                  NODE                    DESIRED STATE       CURRENT STATE              ERROR               PORTS
uqsj8mim0sac        getstartedlab_web.1   friendlyhello:latest   localhost.localdomain   Running             Preparing 3 seconds ago
shjiwlnj12sp        getstartedlab_web.2   friendlyhello:latest   php                     Running             Preparing 24 seconds ago
8sqllvgid8jp        getstartedlab_web.3   friendlyhello:latest   php                     Running             Preparing 24 seconds ago
v7fsecgcg504        getstartedlab_web.4   friendlyhello:latest   php                     Running             Preparing 24 seconds ago
np8utmyvk5px        getstartedlab_web.5   friendlyhello:latest   localhost.localdomain   Running             Preparing 3 seconds ago
[root@localhost compose]$
```

在第二台的worker节点上执行命令会提示失败:

```
[root@php compose]$docker stack ps getstartedlab
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
[root@php compose]$
```

现在,在两台服务器上都能访问刚才部署的app

```
 huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.12:4000
<h3>Hello World!</h3><b>Hostname:</b> 926f433b3896<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>% 

huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.50:4000
<h3>Hello World!</h3><b>Hostname:</b> 1535f17586ea<br/><b>Visits:</b> <i>cannot connect to Redis, counter disabled</i>%                                                            huangyong@huangyong-Macbook-Pro  ~ 
```

#### 扩展app

扩展app还是直接编辑docker-compose.yml文件.然后重新docker stack deploy部署即可.

如果是需要将其他虚拟机或者物理服务器加入进swarm集群,就像第二台服务器一样使用docker swarm join命令加入即可,

#### 停止swarm

命令:

```
docker stack rm getstartedlab
```

#### 总结

```
# 初始化一个swarm集群

docker swarm init --advertise-addr IP

#加入到swarm集群
docker swarm join --token <token> <swarm manager IP>:<port>

#部署app
docker stack deploy -c docker-compose.yml <services name>
> note:在所有docker服务器节点上都需要有docker-compose.yml文件和相关镜像

# 查看services 
docker stack ps <services name>
docker services ls
docker stack ls

# 从swarm集群中删除 services

docker stack rm <service name>

# 删除swarm集群节点

docker swarm leave #worker节点
docker swarm leave --force #manager节点
```

