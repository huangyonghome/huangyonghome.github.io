---
title: docker学习笔记---学习Docker Stack
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## 学习Docker Stack

#### 介绍

一个stack是一组共享依赖包的多个相关的services,并且可以编排和扩展.其实从第4小节开始,在利用compose文件部署app时,就已经开始一直使用stack.但是还只是运行在一个单一服务器的单一service.
现在,你可以学习在多个服务器上,运行多个相关的services.

---

* 使用下面的docker-compose.yml文件替换第4小节中的docker-compose.yml

```
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: ianch/friendlyhello:v1
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
  Visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet
networks:
  webnet:
```

<!--more-->

docker-compose文件稍微做了点改动.添加一个Visualizer服务,placement指令确保这个Visualizer服务仅仅运行在swarm manager节点.

---

#### 部署compose文件

* 初始化swarm

```
docker swarm init

```

* 第二台服务器加入swarm集群

```
[root@php compose]$docker swarm join --token SWMTKN-1-5qr6e90o52h5licxatuvmft65kji5qf1roujebf16auoe5xgam-3d0fuzr8818r6330n88dm1fcu 10.0.0.50:2377
This node joined a swarm as a worker.
```

* 部署app

```
[root@localhost compose]$docker stack deploy -c docker-compose.yml getstartedlab
Creating network getstartedlab_webnet
Creating service getstartedlab_web
Creating service getstartedlab_Visualizer
[root@localhost compose]$
```
> 添加了2个服务.web和Visualizer


```
[root@localhost compose]$docker stack ps getstartedlab
ID                  NAME                         IMAGE                             NODE                    DESIRED STATE       CURRENT STATE                ERROR               PORTS
ds77cn4hgd9w        getstartedlab_web.1          friendlyhello:latest              php                     Running             Running about a minute ago
agzj3veqno63        getstartedlab_Visualizer.1   dockersamples/visualizer:stable   localhost.localdomain   Running             Running about a minute ago
3hq0if79g3gk        getstartedlab_web.2          friendlyhello:latest              php                     Running             Running about a minute ago
iuxh7qjfpikw        getstartedlab_web.3          friendlyhello:latest              localhost.localdomain   Running             Running 53 seconds ago
ba9hcq5zmlbd        getstartedlab_web.4          friendlyhello:latest              php                     Running             Running about a minute ago
o600yqmqets7        getstartedlab_web.5          friendlyhello:latest              localhost.localdomain   Running             Running 55 seconds ago
[root@localhost compose]$
```

访问任意一台服务器的8080端口,可以看到Visualizer服务正在运行

![](https://docs.docker.com/get-started/images/get-started-visualizer1.png)

> 这是我借用的官网的图片.

可以看到,visualizer运行在swarm manager节点上,5个web服务运行在swarm集群上.visualizer是一个不需要任何依赖,而可以运行在任何app的独立服务.现在尝试一下创建一个具有依赖项的服务:提供访问计数器的Redis服务

---

### 编辑docker-compose文件

```
version: "3"
services:
  web:
    # replace username/repo:tag with your name and image details
    image: ianch/friendlyhello:v1
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: "0.1"
          memory: 50M
      restart_policy:
        condition: on-failure
    ports:
      - "4000:80"
    networks:
      - webnet
  Visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet

  redis:
    image: redis
    ports:
      - "6379:6379"
    volumes:
      - "/home/docker/data:/data"
    deploy:
      placement:
        constraints: [node.role == manager]
    command: redis-server --appendonly yes
    networks:
      - webnet
networks:
  webnet:
 
```
 这里我们添加了一个redis服务.在Docker HUB上有redis官方镜像,并且已经暴露了6379端口.所以这里只需要指定redis镜像即可..同样redis也只运行在manager节点服务器.

 这里为了持久化数据,在启动redis容器的时候指定了appendonly参数,并且挂载了本机的/home/docker/data目录映射到容器的/data.(redis容器默认保存数据路径)

 * 在manager节点创建/home/docker/data目录

```
[root@localhost compose]$mkdir -pv /home/docker/data
mkdir: 已创建目录 "/home/docker"
mkdir: 已创建目录 "/home/docker/data"
```

* 部署compose

```
[root@localhost compose]$docker stack deploy -c docker-compose.yml getstartedlab
Updating service getstartedlab_web (id: mtgafxttekwfh0tkhkaespa1v)
image friendlyhello:latest could not be accessed on a registry to record
its digest. Each node will access friendlyhello:latest independently,
possibly leading to different nodes running different
versions of the image.

Updating service getstartedlab_Visualizer (id: zmim1kj44afsr9xay8ppxker6)
Creating service getstartedlab_redis
```

可以看到3个services都启动起来了

```
[root@localhost compose]$docker service ls
ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
zmim1kj44afs        getstartedlab_Visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
elomaiu5go9p        getstartedlab_redis        replicated          1/1                 redis:latest                      *:6379->6379/tcp
mtgafxttekwf        getstartedlab_web          replicated          5/5                 friendlyhello:latest              *:4000->80/tcp
```

在浏览器访问服务器的4000端口可以看到有一个访问计数器在增加

```
 huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.50:4000
<h3>Hello World!</h3><b>Hostname:</b> eed56ca4164b<br/><b>Visits:</b> 5%                                                                                                           huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.50:4000
<h3>Hello World!</h3><b>Hostname:</b> 777d2cab6468<br/><b>Visits:</b> 6%                                                                                                           huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.50:4000
<h3>Hello World!</h3><b>Hostname:</b> 213e6a729c6a<br/><b>Visits:</b> 7%                                                                                                           huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.50:4000
<h3>Hello World!</h3><b>Hostname:</b> 85ccc6b1cb18<br/><b>Visits:</b> 8%                                                                                                           huangyong@huangyong-Macbook-Pro  ~ 
```

访问另外一台服务器也可以看到同样结果


```
 huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.12:4000
<h3>Hello World!</h3><b>Hostname:</b> e77e36db18be<br/><b>Visits:</b> 10%                                                                                                          huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.50:4000
<h3>Hello World!</h3><b>Hostname:</b> eed56ca4164b<br/><b>Visits:</b> 11%                                                                                                          huangyong@huangyong-Macbook-Pro  ~  curl http://10.0.0.12:4000
<h3>Hello World!</h3><b>Hostname:</b> eed56ca4164b<br/><b>Visits:</b> 12%
```

访问visulizer容器的8080端口,可以看到redis服务运行

![](https://docs.docker.com/get-started/images/visualizer-with-redis.png)

---

### 管理命令

使用docker node ls 列出swarm集群的所有节点
使用docker service ls 列出所有服务
docker service ps <service_name> 列出某个服务的所有tasks


```
[root@localhost ~]$docker node ls
ID                            HOSTNAME                STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
056r5hr2xb6jjzmbw64m3btd2 *   localhost.localdomain   Ready               Active              Leader              18.09.2
r58bkpsxgi4mjaxrn8octuxw1     php                     Ready               Active                                  18.09.3
[root@localhost ~]$docker service ls
ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
zmim1kj44afs        getstartedlab_Visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
elomaiu5go9p        getstartedlab_redis        replicated          1/1                 redis:latest                      *:6379->6379/tcp
mtgafxttekwf        getstartedlab_web          replicated          5/5                 friendlyhello:latest              *:4000->80/tcp

[root@localhost ~]$docker service ps getstartedlab_web
ID                  NAME                  IMAGE                  NODE                    DESIRED STATE       CURRENT STATE          ERROR               PORTS
ds77cn4hgd9w        getstartedlab_web.1   friendlyhello:latest   php                     Running             Running 14 hours ago
3hq0if79g3gk        getstartedlab_web.2   friendlyhello:latest   php                     Running             Running 14 hours ago
iuxh7qjfpikw        getstartedlab_web.3   friendlyhello:latest   localhost.localdomain   Running             Running 14 hours ago
ba9hcq5zmlbd        getstartedlab_web.4   friendlyhello:latest   php                     Running             Running 14 hours ago
o600yqmqets7        getstartedlab_web.5   friendlyhello:latest   localhost.localdomain   Running             Running 14 hours ago
[root@localhost ~]$
```
