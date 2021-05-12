---
title: Docker原理----3.mount挂载
date: 2021-03-20 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## Docker原理----3.mount挂载

### 介绍

而正如我前面所说的，Namespace 的作用是“隔离”，它让应用进程只能看到该 Namespace 内的“世界”；而 Cgroups 的作用是“限制”，它给这个“世界”围上了一圈看不见的墙。这么一折腾，进程就真的被“装”在了一个与世隔绝的房间里，而这些房间就是 PaaS 项目赖以生存的应用“沙盒”。

可是，还有一个问题不知道你有没有仔细思考过：这个房间四周虽然有了墙，但是如果容器进程低头一看地面，又是怎样一副景象呢？

换句话说，**容器里的进程看到的文件系统又是什么样子的呢？**

<!--more-->

### chroot

Docker容器借助`chroot`挂载一个虚拟根目录到容器.我们在Linux操作系统里可以很方便的演练`chroot`是如何工作的.chroot的作用就是帮助你`change root file system` ,即改变进程的根目录到你指定的位置.



假设，我们现在有一个 $HOME/test 目录，想要把它作为一个 /bin/bash 进程的根目录。

首先，创建一个 test 目录和几个 lib 文件夹:

```
$ mkdir -p $HOME/test
$ mkdir -p $HOME/test/{bin,lib64,lib}
```

然后，把 bash 命令拷贝到 test 目录对应的 bin 路径下

```
cp -v /bin/{bash,ls} $HOME/test/bin
```

接下来，把 bash 命令需要的所有 so 文件，也拷贝到 test 目录对应的 lib 路径下。找到 so 文件可以用 ldd 命令

```
$ T=$HOME/virtual_root
$ list="$(ldd /bin/ls | egrep -o '/lib.*\.[0-9]')"
$ for i in $list; do cp -v "$i" "${T}${i}"; done

#还需要拷贝下面这个so文件
cp -v /usr/lib64/libtinfo.so.5 $T/lib64/
```

最后，执行 chroot 命令，告诉操作系统，我们将使用 $HOME/test 目录作为 /bin/bash 进程的根目录

```
chroot $HOME/test /bin/bash
```

这时，你如果执行 "ls /"，就会看到，它返回的都是 $HOME/test 目录下面的内容，而不是宿主机的内容。

更重要的是，对于被 chroot 的进程来说，它并不会感受到自己的根目录已经被“修改”成 $HOME/test 了。

```
[root@docker-dev test]# chroot $HOME/test /bin/bash
bash-4.2# ./bin/ls /
bin  lib  lib64

bash-4.2# ./bin/ls
bin  lib  lib64
```

这种视图被修改的原理，是不是跟我之前介绍的 Linux Namespace 很类似呢？

---

### rootfs

**实际上，Mount Namespace 正是基于对 chroot 的不断改良才被发明出来的，它也是 Linux 操作系统里的第一个 Namespace。**

当然，为了能够让容器的这个根目录看起来更“真实”，我们一般会在这个容器的根目录下挂载一个完整操作系统的文件系统，比如 Ubuntu16.04 的 ISO。这样，在容器启动之后，我们在容器里通过执行 "ls /" 查看根目录下的内容，就是 Ubuntu 16.04 的所有目录和文件。

**而这个挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，就是所谓的“容器镜像”。它还有一个更为专业的名字，叫作：rootfs（根文件系统）。**

所以，一个最常见的 rootfs，或者说容器镜像，会包括如下所示的一些目录和文件，比如 /bin，/etc，/proc 等等：

```
$ ls /
bin dev etc home lib lib64 mnt opt proc root run sbin sys tmp usr var
```

现在，你应该可以理解，对 Docker 项目来说，它最核心的原理实际上就是为待创建的用户进程：

1. 启用 Linux Namespace 配置；
2. 设置指定的 Cgroups 参数； 
3. 切换进程的根目录（Change Root）。

这样，一个完整的容器就诞生了。不过，Docker 项目在最后一步的切换上会优先使用 pivot_root 系统调用，如果系统不支持，才会使用 chroot。

另外，**需要明确的是，rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。**

所以说，rootfs 只包括了操作系统的“躯壳”，并没有包括操作系统的“灵魂”。

那么，对于容器来说，这个操作系统的“灵魂”又在哪里呢？

实际上，同一台机器上的所有容器，都共享宿主机操作系统的内核。

这就意味着，如果你的应用程序需要配置内核参数、加载额外的内核模块，以及跟内核进行直接的交互，你就需要注意了：这些操作和依赖的对象，都是宿主机操作系统的内核，它对于该机器上的所有容器来说是一个“全局变量”，牵一发而动全身。

这也是容器相比于虚拟机的主要缺陷之一：毕竟后者不仅有模拟出来的硬件机器充当沙盒，而且每个沙盒里还运行着一个完整的 Guest OS 给应用随便折腾。

不过，**正是由于 rootfs 的存在，容器才有了一个被反复宣传至今的重要特性：一致性。**

什么是容器的“一致性”呢？

过去由于云端与本地服务器环境不同，应用的打包过程，一直是使用 PaaS 时最“痛苦”的一个步骤。

但有了容器之后，更准确地说，有了容器镜像（即 rootfs）之后，这个问题被非常优雅地解决了。

**由于 rootfs 里打包的不只是应用，而是整个操作系统的文件和目录，也就意味着，应用以及它运行所需要的所有依赖，都被封装在了一起。**

有了容器镜像“打包操作系统”的能力，这个最基础的依赖环境也终于变成了应用沙盒的一部分。这就赋予了容器所谓的一致性：无论在本地、云端，还是在一台任何地方的机器上，用户只需要解压打包好的容器镜像，那么这个应用运行所需要的完整的执行环境就被重现出来了。

**这种深入到操作系统级别的运行环境一致性，打通了应用在本地开发和远端执行环境之间难以逾越的鸿沟。**

不过，这时你可能已经发现了另一个非常棘手的问题：难道我每开发一个应用，或者升级一下现有的应用，都要重复制作一次 rootfs 吗？

比如，我现在用 Ubuntu 操作系统的 ISO 做了一个 rootfs，然后又在里面安装了 Java 环境，用来部署我的 Java 应用。那么，我的另一个同事在发布他的 Java 应用时，显然希望能够直接使用我安装过 Java 环境的 rootfs，而不是重复这个流程。

一种比较直观的解决办法是，我在制作 rootfs 的时候，每做一步“有意义”的操作，就保存一个 rootfs 出来，这样其他同事就可以按需求去用他需要的 rootfs 了。

但是，这个解决办法并不具备推广性。原因在于，一旦你的同事们修改了这个 rootfs，新旧两个 rootfs 之间就没有任何关系了。这样做的结果就是极度的碎片化。

那么，既然这些修改都基于一个旧的 rootfs，我们能不能以增量的方式去做这些修改呢？这样做的好处是，所有人都只需要维护相对于 base rootfs 修改的增量内容，而不是每次修改都制造一个“fork”。

答案当然是肯定的。

这也正是为何，Docker 公司在实现 Docker 镜像时并没有沿用以前制作 rootfs 的标准流程，而是做了一个小小的创新：

> Docker 在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量 rootfs。

---

### AUFS

当然，这个想法不是凭空臆造出来的，而是用到了一种叫作联合文件系统（Union File System）的能力。

Union File System 也叫 UnionFS，最主要的功能是将多个不同位置的目录联合挂载（union mount）到同一个目录下。比如，我现在有两个目录 A 和 B，它们分别有两个文件：

```
$ tree
.
├── A
│  ├── a
│  └── x
└── B
  ├── b
  └── x
```

然后，我使用联合挂载的方式，将这两个目录挂载到一个公共的目录 C 上：

```
$ mkdir C
$ mount -t aufs -o dirs=./A:./B none ./C
```

这时，我再查看目录 C 的内容，就能看到目录 A 和 B 下的文件被合并到了一起：

```
$ tree ./C
./C
├── a
├── b
└── x
```

可以看到，在这个合并后的目录 C 里，有 a、b、x 三个文件，并且 x 文件只有一份。这，就是“合并”的含义。此外，如果你在目录 C 里对 a、b、x 文件做修改，这些修改也会在对应的目录 A、B 中生效。

---

### Overlay2

AUFS是最古老的联合挂载文件系统,也是docker最初使用的文件系统.在新版本的docker中.使用的是overlay2文件系统.也是目前docker场景下性能最优秀的文件系统.overlay2也是在AUFS的基础之上发展而来.其原理和AUFS有相似之处.下面演示一下overlay2文件系统的用法

* 当前我准备了5个目录.其中A,B每个目录下都有个a文件,其内容如下

```
[root@docker-dev ~]# tree -L 2 A B C D worker
A
├── a
└── x
B
├── a
├── b
└── x
C
D
worker

[root@docker-dev ~]# cat A/a
hello A haha
[root@docker-dev ~]# cat B/a
hello B
```

和AUFS的工作方式类似,将ABC挂载到D这个目录下,其中A,B是底层不可修改目录,C是可读写目录

```
[root@docker-dev ~]# mount -t overlay overlay -o lowerdir=A:B,upperdir=C,workdir=worker D

[root@docker-dev ~]# mount | grep overlay
overlay on /root/D type overlay (rw,relatime,lowerdir=A:B,upperdir=C,workdir=worker)
```

再次查看目录结构,发现D目录下多了三个从目录A和目录B合并过来的a,b,x文件

```
[root@docker-dev ~]# tree -L 2 A B C D worker
A
├── a
└── x
B
├── a
├── b
└── x
C
D
├── a
├── b
└── x
worker
└── work
```

此时在挂载的D目录下,创建一个文件y.修改a文件

```
[root@docker-dev ~]# cd D
[root@docker-dev D]# touch y
[root@docker-dev ~]# echo "hello from D" > D/a

```

再次观察目录结构,发现新增或者修改的a和y文件出现在C目录下,A和B目录保持不变

```
A
├── a
└── x
B
├── a
├── b
└── x
C
├── a
└── y
D
├── a
├── b
├── x
└── y
worker
└── work

```

查看A,B,C目录下的文件a的内容.发现A和B目录下的a文件内容不变,在挂载目录D下修改的a文件"出现"在C目录.

docker镜像就是使用了联合挂载的原理.镜像层类似于目录A和B,他们是不可写的,所有的写操作都发生在类似于目录C的可读写层..下面拿一个容器来举个例子

* 启动一个容器

```
[root@docker-dev ~]# docker run -d --name c1  centos-demo sleep 999
```

* 查看该容器的存储信息.可以看到有2个底层(lowerDir层),以及一个init层.还有一个可读写的upperdir层

```
  "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c-init/diff:/var/lib/docker/overlay2/3c7dd67b43127ddc920230024b1c6c7ead18144cd7ed16d44b7093d9e73fb136/diff:/var/lib/docker/overlay2/53f22fea6b813c35a5356493a840fb550fe75593174bc192446224a9bd0dddbd/diff",
                "MergedDir": "/var/lib/docker/overlay2/e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c/merged",
                "UpperDir": "/var/lib/docker/overlay2/e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c/diff",
                "WorkDir": "/var/lib/docker/overlay2/e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c/work"
            },
            "Name": "overlay2"

```



为什么这里的lowerdir是2层呢? 因为该容器使用的centos-demo镜像就是一个2个layer组成的镜像.通过下面命令可以查看镜像的相关信息:

```
[root@docker-dev ~]# docker inspect centos-demo
 "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:174f5685490326fc0a1c0f5570b8663732189b327007e47ff13d2ca59673db02",
                "sha256:70d298037a86080efd38fbbd051a03c9708b92522cfb58aeef2ed21429ef9b22"
            ]
        },

```

该镜像的Dockerfile文件显示,该镜像只由两个指令组成: FROM和RUN.所以使用`docker build`命令编译成镜像后,该镜像只包含2个layer

```
[root@docker-dev ~]# cat centos/Dockerfile
FROM centos:7
RUN yum update -y \
    && yum install -y net-tools \
       tcpdump \
       bind-utils \
    && yum clean all \
    && rm -rf /var/cache/yum/*

```

回到容器本身.从`docker inspect c1`命令的结果可以看到该容器的各个layer的信息.进入到`/var/lib/docker/overlay2/` 目录下

```
[root@docker-dev ~]# cd /var/lib/docker/overlay2/
```

下面这个子目录显示的就是`FROM centos:7` 指令中基础镜像`centos:7`的文件系统

```
[root@docker-dev overlay2]# ls 53f22fea6b813c35a5356493a840fb550fe75593174bc192446224a9bd0dddbd/diff
anaconda-post.log  bin  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

我们来关注一下容器的可读写层.由于容器刚启动,我们没有对容器进行任何变更.所以容器里没有写入任何新数据

```
[root@docker-dev overlay2]# ll e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c/diff
total 0
[root@docker-dev overlay2]#
```

在容器内部的`/tmp`目录下写入一个新的文件.内容为"hello this is c1"

```
[root@docker-dev overlay2]# docker exec -it c1  bash
[root@55771f4dbf23 /]# echo "hello this is c1" > /tmp/c1.txt
[root@55771f4dbf23 /]# cat /tmp/c1.txt
hello this is c1
```

再次查看宿主机上该容器的可读写层目录

```
[root@docker-dev overlay2]# ll e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c/diff/
total 0
drwxrwxrwt 2 root root 20 May  9 22:55 tmp

[root@docker-dev overlay2]# cat e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c/diff/tmp/c1.txt
hello this is c1
```

从上面的例子中我们可以看到**容器的 rootfs 由如下图所示的三部分组成：**

![image-20210509225741148](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20210509225741148.png)

> 图是盗来的,非本例子中的实际layer信息,但是大同小异

**第一部分，只读层。**

它是这个容器的 rootfs 最下面的五层，对应的正是 `centos:7` 镜像的2层。可以看到，它们的挂载方式都是只读的

**第二部分，可读写层。**

它是这个容器的 rootfs 最上面的一层（e03a913bf2e0ed33），它的挂载方式为：rw，即 read write。在没有写入文件之前，这个目录是空的。而一旦在容器里做了写操作，你修改产生的内容就会以增量的方式出现在这个层中。

可是，你有没有想到这样一个问题：如果我现在要做的，是删除只读层里的一个文件呢？

为了实现这样的删除操作，overlay会在可读写层创建一个 whiteout 文件，把只读层里的文件“遮挡”起来。

比如，你要删除只读层里一个名叫 foo 的文件，那么这个删除操作实际上是在可读写层创建了一个名叫.wh.foo 的文件。这样，当这两个层被联合挂载之后，foo 文件就会被.wh.foo 文件“遮挡”起来，“消失”了。这个功能，就是“ro+wh”的挂载方式，即只读 +whiteout 的含义。我喜欢把 whiteout 形象地翻译为：“白障”。

所以，最上面这个可读写层的作用，就是专门用来存放你修改 rootfs 后产生的增量，无论是增、删、改，都发生在这里。而当我们使用完了这个被修改过的容器之后，还可以使用 docker commit 和 push 指令，保存这个被修改过的可读写层，并上传到 Docker Hub 上，供其他人使用；而与此同时，原先的只读层里的内容则不会有任何变化。这，就是增量 rootfs 的好处。



#### init层

它是一个以“-init”结尾的层，夹在只读层和读写层之间。Init 层是 Docker 项目单独生成的一个内部层，专门用来存放 /etc/hosts、/etc/resolv.conf 等信息。

需要这样一层的原因是，这些文件本来属于只读的 Ubuntu 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 hostname，所以就需要在可读写层对它们进行修改。

可是，这些修改往往只对当前的容器有效，我们并不希望执行 docker commit 时，把这些信息连同可读写层一起提交掉。

所以，Docker 做法是，在修改了这些文件之后，以一个单独的层挂载了出来。而用户执行 docker commit 只会提交可读写层，所以是不包含这些内容的。

```
[root@docker-dev overlay2]# ll e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c-init
total 8
-rw------- 1 root root  0 May  9 22:40 committed
drwxr-xr-x 4 root root 46 May  9 22:40 diff
-rw-r--r-- 1 root root 26 May  9 22:40 link
-rw-r--r-- 1 root root 57 May  9 22:40 lower
drwx------ 3 root root 18 May  9 22:40 work
[root@docker-dev overlay2]# ll e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c-init/diff
total 0
drwxr-xr-x 4 root root 43 May  9 22:40 dev
drwxr-xr-x 2 root root 66 May  9 22:40 etc
[root@docker-dev overlay2]# ll e03a913bf2e0ed33012a912a5ae421e9d0ededc262c998656d828f8cd4a65e4c-init/diff/etc
total 0
-rwxr-xr-x 1 root root  0 May  9 22:40 hostname
-rwxr-xr-x 1 root root  0 May  9 22:40 hosts
lrwxrwxrwx 1 root root 12 May  9 22:40 mtab -> /proc/mounts
-rwxr-xr-x 1 root root  0 May  9 22:40 resolv.conf

```



## 总结

在今天的分享中，我着重介绍了 Linux 容器文件系统的实现方式。而这种机制，正是我们经常提到的容器镜像，也叫作：rootfs。它只是一个操作系统的所有文件和目录，并不包含内核，最多也就几百兆。而相比之下，传统虚拟机的镜像大多是一个磁盘的“快照”，磁盘有多大，镜像就至少有多大。

通过结合使用 Mount Namespace 和 rootfs，容器就能够为进程构建出一个完善的文件系统隔离环境。当然，这个功能的实现还必须感谢 chroot 和 pivot_root 这两个系统调用切换进程根目录的能力。

而在 rootfs 的基础上，Docker 公司创新性地提出了使用多个增量 rootfs 联合挂载一个完整 rootfs 的方案，这就是容器镜像中“层”的概念。

通过“分层镜像”的设计，以 Docker 镜像为核心，来自不同公司、不同团队的技术人员被紧密地联系在了一起。而且，由于容器镜像的操作是增量式的，这样每次镜像拉取、推送的内容，比原本多个完整的操作系统的大小要小得多；而共享层的存在，可以使得所有这些容器镜像需要的总空间，也比每个镜像的总和要小。这样就使得基于容器镜像的团队协作，要比基于动则几个 GB 的虚拟机磁盘镜像的协作要敏捷得多。

更重要的是，一旦这个镜像被发布，那么你在全世界的任何一个地方下载这个镜像，得到的内容都完全一致，可以完全复现这个镜像制作者当初的完整环境。这，就是容器技术“强一致性”的重要体现。



### 参考资料

Cgroup: https://coolshell.cn/articles/17049.html (这位大佬的很多文章都值得学习)
理解overlay2: https://www.cnblogs.com/jiangbo44/p/14056898.html