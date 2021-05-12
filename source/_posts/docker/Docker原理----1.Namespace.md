---
title: Docker原理----1.Namespace
date: 2021-03-20 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## Docker原理----Namespace

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



### docker exec

我们知道通过`docker exec`命令可以进入一个容器的内部.在了解了 Linux Namespace 的隔离机制后，你应该会很自然地想到一个问题：docker exec 是怎么做到进入容器里的呢？

实际上，Linux Namespace 创建的隔离空间虽然看不见摸不着，但一个进程的 Namespace 信息在宿主机上是确确实实存在的，并且是以一个文件的方式存在。

该命令的原理其实就是获取当前容器进程的PID,以及namespace文件.然后加入该namespace.这样通过共享同一个namespace.从而获取容器内部的信息.

下面通过一个示例演示如何进入一个容器内部

* 启动一个临时容器

```
docker run -d --name c1 centos-demo sleep 999
```

* 查看容器进程的PID

```
[root@docker-dev ~]# docker inspect c1 -f {{.State.Pid}}
22733
```

* 查看该进程的namespace

```
[root@docker-dev ~]# ll /proc/22733/ns
total 0
lrwxrwxrwx 1 root root 0 May  9 17:15 ipc -> ipc:[4026532170]
lrwxrwxrwx 1 root root 0 May  9 17:15 mnt -> mnt:[4026532168]
lrwxrwxrwx 1 root root 0 May  9 17:14 net -> net:[4026532173]
lrwxrwxrwx 1 root root 0 May  9 17:15 pid -> pid:[4026532171]
lrwxrwxrwx 1 root root 0 May  9 17:15 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 May  9 17:15 uts -> uts:[4026532169]
```

可以看到，一个进程的每种 Linux Namespace，都在它对应的 /proc/[进程号]/ns 下有一个对应的虚拟文件，并且链接到一个真实的 Namespace 文件上。

有了这样一个可以“hold 住”所有 Linux Namespace 的文件，我们就可以对 Namespace 做一些很有意义事情了，比如：加入到一个已经存在的 Namespace 当中。

**这也就意味着：一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的，这正是 docker exec 的实现原理。**

而这个操作所依赖的，乃是一个名叫 setns() 的 Linux 系统调用。它的调用方法，我可以用如下一段小程序为你说明：

```
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>
 
#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)
 
int main(int argc, char *argv[]) {
    int fd;
    
    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit("setns");
    }
    execvp(argv[2], &argv[2]); 
    errExit("execvp");
}
```

这段代码功能非常简单：它一共接收两个参数，第一个参数是 argv[1]，即当前进程要加入的 Namespace 文件的路径，比如 /proc/22733/ns/net；而第二个参数，则是你要在这个 Namespace 里运行的进程，比如 /bin/bash。

这段代码的的核心操作，则是通过 open() 系统调用打开了指定的 Namespace 文件，并把这个文件的描述符 fd 交给 setns() 使用。在 setns() 执行后，当前进程就加入了这个文件对应的 Linux Namespace 当中了。

现在，你可以编译执行一下这个程序，加入到容器进程（PID=22733）的net也就是 Network Namespace 中：

```
[root@docker-dev ~]#gcc -o set_ns set_ns.c 
[root@docker-dev ~]#./set_ns /proc/22733/ns/net /bin/bash 
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02  
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10 errors:0 dropped:0 overruns:0 carrier:0
	   collisions:0 txqueuelen:0 
          RX bytes:976 (976.0 B)  TX bytes:796 (796.0 B)
 
lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	  collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

正如上所示，当我们执行 ifconfig 命令查看网络设备时，我会发现能看到的网卡“变少”了：只有两个。而我的宿主机则至少有四个网卡。这是怎么回事呢？

实际上，在 setns() 之后我看到的这两个网卡，正是我在前面启动的 Docker 容器里的网卡。也就是说，我新创建的这个 /bin/bash 进程，由于加入了该容器进程（PID=22733）的 Network Namepace，它看到的网络设备与这个容器里是一样的，即：/bin/bash 进程的网络设备视图，也被修改了。

而一旦一个进程加入到了另一个 Namespace 当中，在宿主机的 Namespace 文件上，也会有所体现。

在宿主机上，你可以用 ps 指令找到这个 set_ns 程序执行的 /bin/bash 进程，其真实的 PID 是 22821：

```
[work@docker-dev ~]$ ps aux | grep "/bin/bash"
root     22821  0.0  0.0 115676  2084 pts/0    S+   17:20   0:00 /bin/bash
```

这时，如果按照前面介绍过的方法，查看一下这个 PID=22733的进程的 Namespace，你就会发现这样一个事实：

```
root@docker-dev ~]# ll /proc/22733/ns/net
lrwxrwxrwx 1 root root 0 May  9 17:14 /proc/22733/ns/net -> net:[4026532173]
[root@docker-dev ~]# ll /proc/22821/ns/net
lrwxrwxrwx 1 root root 0 May  9 17:25 /proc/22821/ns/net -> net:[4026532173]
```

在 /proc/[PID]/ns/net 目录下，这个 PID=22821进程，与我们前面的 Docker 容器进程（PID=22733）指向的 Network Namespace 文件完全一样。这说明这两个进程，共享了这个名叫 net:[4026532173] 的 Network Namespace。

刚才我们演示了共享了net这个网络名称空间,同样的道理,还可以共享其他名称空间.比如:

```
[root@docker-dev ~]#  ./set_ns /proc/22733/ns/mnt /bin/bash
[root@docker-dev /]# ls ~
anaconda-ks.cfg
[root@docker-dev /]# exit
exit
[root@docker-dev ~]# ls
1.txt            centos          hsq-openapi-php              memory.limit_in_bytez~  ns.c            set_ns    tradecenter-nginx-php-fpm  zookeeper-3.4.13.tar.gz
2.txt            cert            jdk-8u281-linux-i586.tar.gz  mem.py                  php             set_ns.c  virtual_root
A                D               jenkins-fpm.txt              nginx                   php-fpm         tasks~    worker
anaconda-ks.cfg  Dockerfile      jenkins-php-rpc.txt          nginx-php               php-rpc         tasky~    www.conf
B                Dockerfile.bak  jenkins-slave                nginx-php-rpc           php-rpc:7.1.33  taskz~    yar-2.0.5.tgz
C                fpm.txt         memory.limit_in_bytes~       ns                      php-rpc.txt     test      zookeeper-0.6.4.tgz


```

当进入容器进程的mnt名称空间后,进程视图切换到了`/`根目录下,并且`root`家目录的内容也发生了变化.

以上就是`docker exec`命令背后的详细原理.

---





**总结**

`Namespace`的视图隔离就是Linux容器最基本的实现原理.所以Docker实际上是在创建容器进程时，指定了这个进程所需要启用的一组 Namespace 参数。这样，容器就只能“看”到当前 Namespace 所限定的资源、文件、设备、状态，或者配置。而对于宿主机以及其他不相关的程序，它就完全看不到了。

**所以说，容器，其实是一种特殊的进程而已。**

在理解了 Namespace 的工作方式之后，你就会明白，跟真实存在的虚拟机不同，在使用 Docker 的时候，并没有一个真正的“Docker 容器”运行在宿主机里面。Docker 项目帮助用户启动的，还是原来的应用进程，只不过在创建这些进程时，Docker 为它们加上了各种各样的 Namespace 参数。

这时，这些进程就会觉得自己是各自 PID Namespace 里的第 1 号进程，只能看到各自 Mount Namespace 里挂载的目录和文件，只能访问到各自 Network Namespace 里的网络设备，就仿佛运行在一个个“容器”里面，与世隔绝。



### 思考题

Q: 你是否知道最新的 Docker 项目默认会为容器启用哪些 Namespace 吗？

A: PID, UTS, network, user, mount, IPC, cgroup