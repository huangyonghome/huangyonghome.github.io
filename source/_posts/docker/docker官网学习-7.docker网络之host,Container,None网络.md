---
title: docker学习笔记---docker网络之host,Container,None网络
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker官网学习-7.docker网络之host,Container,None网络

### host网络介绍

如果启动容器的时候使用host模式，那么这个容器将不会获得一个独立的Network Namespace，而是和宿主机共用一个Network Namespace。容器将不会虚拟出自己的网卡，配置自己的IP等，而是使用宿主机的IP和端口。但是，容器的其他方面，如文件系统、进程列表等还是和宿主机隔离的。

---

### 创建host网络

```
docker run -tid --net=host --name busybox busybox1

#host网络下的容器没有虚拟网卡,而是和宿主机共享网络
[root@localhost ~]$docker exec  -it busybox1 ifconfig
docker0   Link encap:Ethernet  HWaddr 02:42:89:26:D1:C9
          inet addr:192.168.1.5  Bcast:192.168.1.255  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1072032 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1997027 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:61915874 (59.0 MiB)  TX bytes:1934238839 (1.8 GiB)
```

> 注:host网络只能工作在Linux主机上.
>
> 如果容器没有暴露任何端口,那host网络没有任何效果

<!--more-->

host网络示意图

![](http://cdn.img2.a-site.cn/pic.php?url=aHR0cDovL21tYml6LnFwaWMuY24vbW1iaXovUVAwQVk3dGRKblV4eFJNWjRRcDl0b21GaFFRMDNYVUVObjRZOVg0Q2tRRVZMV2dFZGt4MWljeThKY3VERXFhTGFZZUhiaDB1TWJZeVVtTjhQQ1l0bDl3LzA/d3hfZm10PXBuZw==)

---

### Container网络介绍

这个模式指定新创建的容器和已经存在的一个容器共享一个 Network Namespace，而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的 IP，而是和一个指定的容器共享 IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。两个容器的进程可以通过 lo 网卡设备通信。

Container网络示意图

![](http://cdn.img2.a-site.cn/pic.php?url=aHR0cDovL21tYml6LnFwaWMuY24vbW1iaXovUVAwQVk3dGRKblV4eFJNWjRRcDl0b21GaFFRMDNYVUVtaWM5elRoU1d1UmdSc2xVT2oyeHpmeUljZXdpY2E3VkpibE03Nnc5N01PZFRLVEl2TkdpYTBPd2cvMD93eF9mbXQ9cG5n)

---

### None网络

使用none模式，Docker容器拥有自己的Network Namespace，但是，并不为Docker容器进行任何网络配置。也就是说，这个Docker容器没有网卡、IP、路由等信息。需要我们自己为Docker容器添加网卡、配置IP等。

Node模式示意图:

![](http://cdn.img2.a-site.cn/pic.php?url=aHR0cDovL21tYml6LnFwaWMuY24vbW1iaXovUVAwQVk3dGRKblV4eFJNWjRRcDl0b21GaFFRMDNYVUVMeFREaWNxUVFYQ0dObVlUNFlRdVdQYkxBRk1TVmhvRFlrcUtENFVLczVXbWtqbTM1THNpY1FZUS8wP3d4X2ZtdD1wbmc=)