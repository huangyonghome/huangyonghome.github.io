---
title: docker学习笔记---Docker镜像篇
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker笔记——镜像篇

### 介绍

docker镜像是一个只读的Docker容器模板.含有启动docker容器所需的文件系统结构以及内容.因此是启动一个容器的基础.docker镜像的文件内容以及一些运行docker容器的配置文件组成了docker容器的静态文件运行环境—rootfs.

可以这么理解,docker镜像是docker容器的静态视角.docker容器是docker镜像的运行状态

---

**1.rootfs**

rootfs是docker容器的根目录.如:/dev,/proc,/bin,/etc …….传统的Linux容器操作系统内核启动时,首先挂载一个只读(read-only)的rootfs.当系统检测到完整性后,再将其切换到读写(read-write)模式.而在docker架构中.也沿用了Linux内核的启动方法.在docker为容器挂载rootfs时,将rootfs设置为只读模式,挂载完毕后,在已有的只读rootfs上再挂载一个读写层.

读写层位于docker容器文件系统的最顶层.下面可能挂载了多个只读层.

<!--more-->

**2.docker镜像的特点**

* **分层**

每个镜像都由一系列的"镜像层"组成.当需要修改容器镜像内的某个文件时,只对最上方的读写层进行修改,不覆盖下面的只读层文件系统.例如删除一个只读文件系统中的文件时,只会在读写层标记这个文件"已经被删除",但是这个文件在只读层中仍然存在.只不过不被用户感知.

* **写时复制(copy-on-write)**

每个容器在启动的时候并不需要单独复制一份镜像文件,而是将所有镜像层以只读的方式挂载到一个挂载点,在多个容器之间共享.在未更改镜像文件内容时,所有容器共享一份数据,只有在docker容器运行过程中修改过文件时,才会把变化的文件内容写到读写层.并隐藏只读层中的老版本文件.

写时复制机制减少了镜像对磁盘空间的占用和容器的启动时间

* **联合挂载**

联合挂载技术可以在一个挂载点同时挂载多个文件系统.实现这种联合挂载技术的文件系统被称为联合文件系统(union filesystem).从内核的角度来看,docker容器的文件系统分为只读rootfs层和读写层.但是在用户的视角看来,整个文件系统都是rootfs底层.

下面这个图可以理解,镜像是由一堆的只读层堆叠起来的统一视角:

![](![Docker镜像](https://img1.jesse.top/docker-image1.gif)



下面这个图理解了docker镜像和docker容器的区别

![](https://img1.jesse.top/docker-container1.png)



---

### docker镜像的相关概念

1.**registry**

registry用来保存docker镜像.可以将registry简单的想象成类似于git仓库之类的实体.当```docker run```命令启动一个容器时,如果宿主机上并不存在该镜像,那么docker将从registry中下载镜像并保存到宿主机

可以使用docker官方的公共registry服务(docker hub),可以可以使用阿里云私有的registry,甚至还可以自己搭建私有的registry

**2.repository**

repository是由具有某个功能的docker镜像的所有迭代版本构成的镜像组.repository通常表示镜像所具有的功能,例如ansible/ubunbu14.4-ansible.而顶层仓库则只包含repository名.例如,Ubuntu

repository是一个镜像集合,包含了多个不同版本的镜像.使用标签进行版本区分,例如ubuntu:14.04,ubuntu12.04.他们均属于ubuntu这个repository

**总而言之,registry是repository的集合,repository是镜像的集合**

---

### docker镜像相关的命令

* **拉取镜像**

```docker pull [OPTIONS] NAME[:TAG|@DIGEST]```

如果只指定了镜像名,则默认从docker hub官方拉取该镜像的最新latest版本

```
[root@localhost ~]$docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
743f2d6c1f65: Pull complete
6bfc4ec4420a: Pull complete
688a776db95f: Pull complete
Digest: sha256:23b4dcdf0d34d4a129755fc6f52e1c6e23bb34ea011b315d87e193033bcd1b68
Status: Downloaded newer image for nginx:latest
```

如果指定了tag,则拉取指定的版本镜像

```
[root@localhost ~]$docker pull nginx:1.15
1.15: Pulling from library/nginx
Digest: sha256:23b4dcdf0d34d4a129755fc6f52e1c6e23bb34ea011b315d87e193033bcd1b68
Status: Downloaded newer image for nginx:1.15
```

拉取我阿里云的私人registry下的镜像

```
#registry.cn-hangzhou.aliyuncs.com/jesse_images为仓库地址
#php7.1.9为镜像版本
[root@localhost ~]$docker pull registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:php7.1.9

php7.1.9: Pulling from jesse_images/jesse_images
Digest: sha256:ed9b7326b539f47a81697e51ed8ec698bec49fb62959990c1277d068fc55ff94
Status: Downloaded newer image for registry.cn-hangzhou.aliyuncs.com/jesse_images/jesse_images:php7.1.9
```

---

* **删除镜像**

命令格式:

```docker rmi [OPTIONS] IMAGE [IMAGE…]```

可以是docker rmi 镜像ID 或者 docker rmi 镜像名:tag

```
docker images #查看当前宿主机上的镜像
[root@localhost ~]$docker images
REPOSITORY                                                    TAG                    IMAGE ID            CREATED             SIZE
busybox                                                       latest                 64f5d945efcc        6 days ago          1.2MB
nginx                                                         1.15                   53f3fd8007f7        7 days ago          109MB
nginx                                                         latest                 53f3fd8007f7        7 days ago          109MB
php-swoole                                                    7.1                    aa71c42a22ca        9 days ago          588MB
<none>                                                        <none>                 01f5d7914e61        9 days ago          585MB

#删除镜像ID为01f5d7914e61的镜像

[root@localhost ~]$docker rmi 01f5d7914e61
Deleted: sha256:01f5d7914e615b0e2f7cc36a494c876dfc0c678963898374d9ef512d7a762aac
Deleted: sha256:b4dd4a057d2561647ff7bf6b299a143c99f66831c129618f49bca5e6ac82f99e
Deleted: sha256:c37c880338efd3d340bfa71b35b7653b6cec8eb4f5dfcfab8c7ad0045fef3ce6
Deleted: sha256:fb7d015f8921c1244134730b6c21f0bda6c7156ccd421d9e0069d5a1074b48dd
Deleted: sha256:ab74760ab0af7680fa9338100c92306392ffeb384b8976045a11dab9a4ebbc57
Deleted: sha256:8544a2552375c861955db9034e9c3c5a3e83530b84de9b9bb6d4a7d0d5e5b8ac
Deleted: sha256:4eebc2d39a0733b28992a064fc71852297927a3994b01a9d1123d71b042ab729

#删除nginx.tag为1.15的镜像
[root@localhost ~]$docker rmi nginx:1.15
Untagged: nginx:1.15
```

---

* **查看镜像**

```docker history 镜像名```可以看到镜像的构建分层

```
[root@localhost ~]$docker history nginx
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
53f3fd8007f7        7 days ago          /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B
<missing>           7 days ago          /bin/sh -c #(nop)  STOPSIGNAL SIGTERM           0B
<missing>           7 days ago          /bin/sh -c #(nop)  EXPOSE 80                    0B
<missing>           7 days ago          /bin/sh -c ln -sf /dev/stdout /var/log/nginx…   0B
<missing>           7 days ago          /bin/sh -c set -x  && apt-get update  && apt…   54.1MB
<missing>           7 days ago          /bin/sh -c #(nop)  ENV NJS_VERSION=1.15.12.0…   0B
<missing>           7 days ago          /bin/sh -c #(nop)  ENV NGINX_VERSION=1.15.12…   0B
<missing>           7 days ago          /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B
<missing>           8 days ago          /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>           8 days ago          /bin/sh -c #(nop) ADD file:fcb9328ea4c115670…   55.3MB
```

```docker inspect 镜像名``` 可以看到镜像的具体信息

---

* **创建镜像**

```docker commit```命令可以基于现有的容器创建出一个镜像

```
#用法格式:
docker commit -m '镜像说明信息'   -a  作者  容器ID 镜像名:版本

[root@localhost ~]$docker commit -h

Usage:	docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]

Create a new image from a container's changes

Options:
  -a, --author string    Author (e.g., "John Hannibal Smith <hannibal@a-team.com>")
  -c, --change list      Apply Dockerfile instruction to the created image
  -m, --message string   Commit message
  -p, --pause            Pause container during commit (default true)
  
  #例如.将正在运行中的Nginx容器提交为一个新的nginx:test镜像
[root@localhost ~]$docker commit -m 'test' -a 'jesse' nginx nginx:test
sha256:028f5e2b21a66a1bf5f70727f20cac04e8918f57d5584cc2aeb09f18791d9680
```

---

* 导入导出镜像

命令: 

```docker save -o 保存文件名 镜像名:tag```  ————将某个镜像保存为一个文件

```docker load < 文件名``` or ```docker load —input 文件名``` ——将某个文件导入到本地镜像

例如

```
#将nginx:test这个镜像保存为Nginx_test.tar文件
[root@localhost ~]$docker save -o nginx_test.tar nginx:test
[root@localhost ~]$ll nginx_test.tar
-rw------- 1 root root 113036800 5月  16 10:29 nginx_test.tar

#删除ningx:test这个镜像.然后再从该文件恢复
[root@localhost ~]$docker load < nginx_test.tar
67392954caf5: Loading layer [==================================================>]  8.192kB/8.192kB
Loaded image: nginx:test

#镜像已经被导入
[root@localhost ~]$docker images | grep nginx
nginx                                                         test                   028f5e2b21a6        4 minutes ago       109MB
```

