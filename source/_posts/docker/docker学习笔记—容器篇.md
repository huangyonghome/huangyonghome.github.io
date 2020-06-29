---
title: docker学习笔记---docker容器篇
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker学习笔记--- docker容器篇

**一.创建并启动容器**

* **创建容器**

```
docker create -it 镜像名:tag
```

* **查看所有容器**(包括运行中,已退出,错误容器)

```
docker ps -a
```

* 查看所有容器的ID

```
docker ps -qa
```

* **启动容器**

```
docker start container_id
```

* **使用指定的镜像直接创建并启动容器**

```
docker run -it 镜像名:标签 COMMAND

jesse@jesse-virtual-machine:~$ docker run -it ubuntu:16.04 /bin/bash
root@66a29f973548:/#   ----------此时已经进入到容器的bash环境
```

<!--more-->

当利用 docker run 来创建容器时，Docker 在后台运行的标准操作包括：

1.检查本地是否存在指定的镜像，不存在就从公有仓库下载

2.利用镜像创建并启动一个容器

3.分配一个文件系统，并在只读的镜像层外面挂载一层可读写层

4.从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去

5.从地址池配置一个 ip 地址给容器

6.执行用户指定的应用程序

7.执行完毕后容器被终止

* **以守护状态运行:**

以守护态运行（加参数-d):

```
docker run -d registry.intra.weibo.com/yushuang3/centos:v1 /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

* **容器终止:**

**获取容器输出的信息**

```
docker logs container_id
```

**停止容器**

```
docker stop container_Id
```

> 终止一个容器  加入-t=10 表示等待10秒(不加-t选项则默认就是10秒)再次发送SIGKILL信号终止容器

**重启容器**

``` 
docker restart container_id
```

**容器状态:**

- **status表示容器的状态..**

* **exited 表示容器已经退出**
* **up 表示容器正在运行**



docker run启动容器时还可以指定其他的配置参数:

* -h HOSTNAME 或者 --hostname=HOSTNAME.设置容器的主机名
* --dns=IP_ADDRESS:设置容器的DNS.写在容器的/etc/resolv.conf文件中.


```
[root@localhost test]$docker run -itd --name busybox1 -h dwd-busybox --dns=8.8.8.8 busybox
c25dac9c641705e00f02aefe302987f39f853a1feb8c0d3f32dc1675747edd84

[root@localhost test]$docker exec -it busybox1 hostname
dwd-busybox

[root@localhost test]$docker exec -it busybox1 cat /etc/resolv.conf
nameserver 8.8.8.8
```
但是需要注意的是.这些修改不会被 docker commit保存,也就是不会保存在镜像中

```
#将busybox1容器保存为busybox1:test镜像
[root@localhost test]$docker commit -m 'test' -a 'jesse' busybox1 busybox1:test
sha256:3c8faee532f177cf8bb8736db89694c2c3ff5be1a30a15d604e450130909d123

#用这个镜像,启动一个新的busybox1-test的容器
[root@localhost test]$docker run --name busybox1-test -itd busybox1:test
9326b615e9e3af64336683f7f82e048929de560d4ad0a5caf2485bbc4a62e18c

#可以看到hostname和dns信息没有被保留
[root@localhost test]$docker exec -it busybox1-test hostname
9326b615e9e3

[root@localhost test]$docker exec -it busybox1-test cat /etc/resolv.conf
nameserver 114.114.114.114
nameserver 114.114.115.115
```
