---
title: Docker原理----1.Namespace
date: 2021-03-20 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## Docker原理----1.Namespace

### 介绍

容器其实是一种沙盒技术,顾名思义,沙盒就像是一个集装箱一样,把你的应用"装"起来.这样应用和应用之间就因为有了边界而不至于互相干扰.而被装进集装箱的应用,也可以被方便的搬来搬去.

容器的核心功能就是通过约束和修改进程的动态表现,从而为其创造出一个"边界".对于Docker以及大多数的Linux容器来说.`cgroup` 是用来约束的手段,而`Namespace`则是用来修改进程视图的主要方法



### Namespace

linux内核提拱了6种namespace隔离的系统调用，如下图所示，但是真正的容器还需要处理许多其他工作。

| namespace | 系统调用参数  | **隔离内容**               |
| --------- | ------------- | -------------------------- |
| UTS       | CLONE_NEWUTS  | 主机名或域名               |
| IPC       | CLONE_NEWIPC  | 信号量、消息队列和共享内存 |
| PID       | CLONE_NEWPID  | 进程编号                   |
| Network   | CLONE_NEWNET  | 网络设备、网络战、端口等   |
| Mount     | CLONE_NEWNS   | 挂载点（文件系统）         |
| User      | CLONE_NEWUSER | 用户组和用户组             |

实际上，linux内核实现namespace的主要目的，就是为了实现轻量级虚拟化技术服务。在同一个namespace下的进程合一感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，仿佛自己置身一个独立的系统环境中，以达到隔离的目的。

下面拿个简单例子来讲解部分Namespace技术

<!--more-->

#### PID namespace

启动一个容器,在容器里`/bin/sh`的进程的PID是1.也就是容器的第一号进程,也称之为容器的主进程.另外还可以注意到该容器只能看到2个进程,一个是启动的PID为1的主进程,一个是手动在容器内部执行的ps进程

```
[work@docker-dev elasticsearch]$ docker run -it busybox /bin/sh
/ # ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    6 root      0:00 ps aux
/ #
```

接下来在Docker宿主机上查找`/bin/sh`这个进程,发现他的PID为8058,而它的父进程是`containerd-shim`,再继续往上找,他的父进程的PID为1571,也就是`/usr/bin/containerd` 

```
[work@docker-dev ~]$ ps -ef | grep "/bin/sh"
work      8024  7016  0 22:09 pts/4    00:00:00 docker run -it busybox /bin/sh
root      8058  8040  0 22:09 pts/0    00:00:00 /bin/sh
work      8118  8094  0 22:10 pts/0    00:00:00 grep --color=auto /bin/sh

[work@docker-dev ~]$ ps -ef | grep 8040
root      8040  1571  0 22:09 ?        00:00:00 containerd-shim -namespace moby -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/moby/17e50d35b5bc523baa1acceb8710accf13b6859365e042278555f336a6642aff -address /run/containerd/containerd.sock -containerd-binary /usr/bin/containerd -runtime-root /var/run/docker/runtime-runc
root      8058  8040  0 22:09 pts/0    00:00:00 /bin/sh
work      8120  8094  0 22:10 pts/0    00:00:00 grep --color=auto 8040

[work@docker-dev ~]$ ps aux | grep 1571 | head -n 1
root      1571  0.0  0.3 1632744 53924 ?       Ssl  Feb01  36:04 /usr/bin/containerd

```

**这说明了什么呢**?

从上面的例子中可以看到Docker容器内的主进程其实是直接运行在Docker宿主机上的.他在宿主机上是个普通的进程,Docker对这个进程实施了一个"障眼法",让他看不到其他的所有进程.让该进程误认为自己是第一个启动的PID为1的进程.

这种技术就是Linux里面的`Namespace`机制.



**总结**

`Namespace`的视图隔离就是Linux容器最基本的实现原理.所以Docker实际上是在创建容器进程时，指定了这个进程所需要启用的一组 Namespace 参数。这样，容器就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置。而对于宿主机以及其他不相关的程序，它就完全看不到了。

**所以说，容器，其实是一种特殊的进程而已。**

在理解了 Namespace 的工作方式之后，你就会明白，跟真实存在的虚拟机不同，在使用 Docker 的时候，并没有一个真正的“Docker 容器”运行在宿主机里面。Docker 项目帮助用户启动的，还是原来的应用进程，只不过在创建这些进程时，Docker 为它们加上了各种各样的 Namespace 参数。

这时，这些进程就会觉得自己是各自 PID Namespace 里的第 1 号进程，只能看到各自 Mount Namespace 里挂载的目录和文件，只能访问到各自 Network Namespace 里的网络设备，就仿佛运行在一个个“容器”里面，与世隔绝。



### 思考题

Q: 你是否知道最新的 Docker 项目默认会为容器启用哪些 Namespace 吗？

A: PID, UTS, network, user, mount, IPC, cgroup