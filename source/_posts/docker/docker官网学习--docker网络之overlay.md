---
title: docker学习笔记---docker网络之overlay
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker官网学习-7.docker网络之overlay

### 介绍

overlay网卡在多个docker宿主机之间创建一个分布式的网络,允许多个容器安全通信.

当初始化一个swarm集群,或者加入docker宿主机到一个swarm集群中.Docker会在该宿主机上创建2个网络:

* ingress: 负责swarm集群的控制以及数据流量
* docker_gwbridge:一个Bridge网络,负责连接swarm集群中的每个docker节点

overlay网络的创建方式和bridge一样.也是docker network create命令

---

### 创建overlay网络

<!--more-->

**创建overlay网络的前提条件**

1.防火墙开通以下端口

* TCP 2377——集群管理节点通信
* TCP,UPD 7946—节点间通信
* UDP 4789—overlay网络流量

2.初始化docker宿主机为swarm集群的manager角色.命令为```docker swarm init```.或者使用```docker swarm join```命令加入到一个现有的swarm集群.

这两种方式都会创建默认的ingress overlay网络.

> 即使你不打算使用swarm服务,但是也要这样做.然后才能创建自定义的overlay网络



**创建overlay网络命令**

```
#创建个overlay网络.名字为my-overlay
docker network create -d overlay my-overlay
```

如果swarm服务或者独立容器需要和其他docker宿主机上的独立容器通信.需要加上```—attachable```参数

```
#创建个overlay网络,名字为my-overlay.并且和其他docker宿主机的standalone容器通信.
docker network create -d overlay --attachable my-overlay
```

> 可以自定义地址段,掩码,网关等信息.具体方法见docker network create --help

---

### 自定义默认Ingress网络

大部分用户不需要配置ingress网络.但是docker17.05以上版本可以自定义ingress网络.如果默认的Ingress网络iP地址段和已经存在的网络有冲突,则自定义的配置可很有用.

配置Ingress网络需要先删除ingress,然后重新创建.所以最好是在创建容器服务之前先定义好ingress.如果已经有暴露出端口的服务.则需要先删除服务.

**自定义默认ingress网络步骤如下:**

1.查看当前ingress网络.```docker network inspect ingress```.删除所有连接到ingress的容器的服务.

2.移除现有的ingress网络.```docker network rm ingress```

3.使用```--ingress```参数重新创建一个overlay网络.定义自定义参数.例如:

```
 docker network create \
  --driver overlay \
  --ingress \
  --subnet=10.11.0.0/16 \
  --gateway=10.11.0.2 \
  --opt com.docker.network.driver.mtu=1200 \
  my-ingress
```

> 可以用自定义名称(my-ingress)来定义ingress网络.但是只允许存在一个自定义Ingress.

4.重启步骤1中的服务

---

### 自定义默认docker_gwbridge 接口

docker_gwbridge是连接overlay网络和docker宿主机物理网卡之间的虚拟网桥.当初始化一个swarm集群,或者将docker宿主机加入到一个swarm集群时,docker会自动创建docker_gwbridge.但是docker_gwbridge不是一个docker服务,而是存在于docker宿主机的内核当中.

所以需要在加入到swarm集群前先配置好docker_gwbridge.

**自定义默认docker_gwbridge网络步骤如下:**

1.停止docker

2.删除当前```docker_gwbridge```

```
$ sudo ip link set docker_gwbridge down

$ sudo ip link del dev docker_gwbridge
```

3.开启docker.不要加入或者初始化swarm

4.手动创建```docker_gwbridge```.下面这个例子定义了网络iP地址范围

> 关于docker_gwbridge的更多参数请参考[Bridg dirver options](<https://docs.docker.com/engine/reference/commandline/network_create/#bridge-driver-options>)

```
$ docker network create \
--subnet 10.11.0.0/16 \
--opt com.docker.network.bridge.name=docker_gwbridge \
--opt com.docker.network.bridge.enable_icc=false \
--opt com.docker.network.bridge.enable_ip_masquerade=true \
docker_gwbridge
```

5.初始化或者加入到swarm集群





