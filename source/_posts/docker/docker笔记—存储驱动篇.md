---
title: docker学习笔记---Docker存储驱动篇
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker笔记——存储驱动篇

### 介绍

通过前一篇的镜像笔记,我们知道docker的镜像是只读的,而且通过同一个镜像启动的docker容器,他们共享同一份底层镜像文件.

这里主要说一说.这些分层的多个只读Image镜像是如何在磁盘中存储的.

---

### docker存储驱动

docker提供了多种存储驱动来实现不同的方式存储镜像，下面是常用的几种存储驱动：

- AUFS
- OverlayFS
- Devicemapper
- Btrfs
- ZFS

下面说一说AUFS、OberlayFS及Devicemapper，更多的存储驱动说明可参考：http://dockone.io/article/1513

<!--more-->

#### AUFS

AUFS（AnotherUnionFS）是一种Union FS，是文件级的存储驱动。AUFS是一个能透明覆盖一个或多个现有文件系统的层状文件系统，把多层合并成文件系统的单层表示。简单来说就是支持将不同目录挂载到同一个虚拟文件系统下的文件系统。这种文件系统可以一层一层地叠加修改文件。无论底下有多少层都是只读的，只有最上层的文件系统是可写的。当需要修改一个文件时，AUFS创建该文件的一个副本，使用CoW(写时复制)将文件从只读层复制到可写层进行修改，结果也保存在可写层。

通常来说最上层是可读写层,下层是只读层.当需要读取一个文件A时,会从最顶层的读写层开始向下寻找.本层没有则根据层关系到下一层开始找.直到找到第一个文件A

当需要写入一个文件A时,如果这个文件不存在,则在读写层新建一个.否则会像上面的步骤一样从顶层开始寻找,找到A文件后,复制到读写层进行修改

当需要删除一个文件A时,如果这个文件仅仅存在读写层,则直接删除.否则就需要先在读写层删除,然后再在读写层创建一个whiteout文件来标志这个文件不存在,而不是真正删除底层文件.

当新建一个文件A.如果这个文件在读写层存在对应的whiteout文件,则先将whiteout文件删除再新建.否则直接读写层新建

在Docker中，底下的只读层就是image，可写层就是Container。结构如下图所示：

![](https://img1.jesse.top/docker-aufs.jpg)



如果你正在使用aufs作为存储驱动,那么在Docker的工作目录(/var/lib/docker)和image下发现aufs目录:

```
root@docker:~# tree /var/lib/docker -d -L 1
/var/lib/docker
├── aufs
├── containers
├── image
├── network
├── plugins
├── swarm
├── tmp
├── trust
└── volumes

root@docker:~# tree /var/lib/docker/image -d -L 1
/var/lib/docker/image
└── aufs

root@docker:~# tree /var/lib/docker/aufs/ -d -L 1
/var/lib/docker/aufs/
├── diff
├── layers
└── mnt
```



在docker工作目录的aufs目录下有3个目录.其中mnt为aufs的挂载目录,diff为实际数据,包括只读层和读写层.layers为每层依赖有关的层描述文件

---

### Device mapper

Device mapper是Linux内核2.6.9后支持的，提供的一种从逻辑设备到物理设备的映射框架机制，在该机制下，用户可以很方便的根据自己的需要制定实现存储资源的管理策略。前面讲的AUFS和OverlayFS都是文件级存储，而Device mapper是块级存储，所有的操作都是直接对块进行操作，而不是文件。

Device mapper驱动会先在块设备上创建一个资源池，然后在资源池上创建一个带有文件系统的基本设备，所有镜像都是这个基本设备的快照，而容器则是镜像的快照。所以在容器里看到文件系统是资源池上基本设备的文件系统的快照，并不有为容器分配空间。当要写入一个新文件时，在容器的镜像内为其分配新的块并写入数据，这个叫用时分配。

当要修改已有文件时，再使用CoW为容器快照分配块空间，将要修改的数据复制到在容器快照中新的块里再进行修改。Device mapper 驱动默认会创建一个100G的文件包含镜像和容器。每一个容器被限制在10G大小的卷内，可以自己配置调整。结构如下图所示：

![](https://img1.jesse.top/docker-devicemapper.jpg)



在Centos 7发行版上最新版的docker中,默认的存储驱动就是device mapper.但是提示已经被弃用了

```
[root@localhost ~]$docker info
Containers: 3
 Running: 3
 Paused: 0
 Stopped: 0
Images: 138
Server Version: 18.09.2
Storage Driver: devicemapper #这一行
......略......
WARNING: the devicemapper storage-driver is deprecated, and will be removed in a future release. #最后这一行提示devicemapper已经被弃用
```

和aufs一样,在docker的工作目录下也能看到device mapper目录

```
[root@localhost ~]$ll /var/lib/docker/devicemapper/
总用量 32
drwx------ 2 root root    32 2月  23 16:25 devicemapper
drwx------ 2 root root 24576 5月  16 10:30 metadata
drwxr-xr-x 5 root root  4096 5月  16 10:30 mnt
```

