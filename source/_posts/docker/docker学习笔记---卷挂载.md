---
title: docker学习笔记---Docker卷挂载
date: 2020-07-04 21:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker学习笔记---卷挂载

### 回顾

回顾一下docker网络的相关知识:

* docker默认所有容器连接到docker0这个bridge虚拟交换机,连接到同一个虚拟交换机的所有容器网络可以互访
* docker可以指定一个bridge虚拟交换机,只需要在`docker run`命令中,使用`--network`参数
* 连接到指定bridge虚拟机交换机的所有容器之间不仅仅可以互访,而且还可以直接访问对方容器的主机名
* 在`docker run`命令中使用`-p`参数可以将容器内部端口映射给外部宿主机,使外部客户端可以访问容器内提供的服务
* 可以使用多个`-p`选项暴露出多个端口,`-p`暴露端口的格式如下:
  - `-p Container_Port` (将宿主机上随机的一个端口,映射到后端指定的端口.例如: -p 80)
  - `-p Host_Port:Container_Port`(这是使用最多的方式,将宿主机上指定的一个端口,映射给容器内某个端口.例如: -p 80:80)
  - `-p range_Port:range_Port`(映射一组连续的端口.例如: -p 1100-1110:1100-1110)
* 无法给运行中的容器,增加端口映射.所以想给当前运行中的容器增加映射端口,只能删除容器,重新run一个
* 端口映射的本质是添加nat转发记录
* 不是所有的容器都需要映射端口.

<!--more-->

---

## 介绍

Docker容器是有生命周期的.一个容器就是一个应用,应用始终有停止和删除的时候.一旦容器被删除,则容器内产生的任何数据都会被删除.

大部分应用都是有状态的服务,比如mysql,redis,nginx(只作为反向代理的Nginx除外),这类容器一旦被删除,新创建的容器无法继续工作.所以任何有状态的应用都需要将数据持久存储.

另外,容器内部都是微型的OS系统,只提供应用程序所必须的依赖环境和进程文件.所以缺少各种调试工具(vim.telnet,ping)等.所以Docker的初衷是把容器看做是一个单一的应用,无需进入容器做任何操作.但是应用的配置文件可能需要经常修改,(比如nginx虚拟主机,php-fpm配置文件等).进入容器内部操作并不是提倡的办法.

---

### Docker 卷介绍

基于以上难题,为了实现有状态应用的持久化数据存储和频繁修改容器内部文件.Docker提供了数据卷(Volume)实现.

卷(Volume)是一个特殊的目录,或者一个特殊的文件系统,将宿主机本地的文件系统(或者目录)映射到一个或者多个容器内部的某个目录.将两者进行绑定,从而实现容器数据持久化.

**Docker卷特性**

* 卷是独立的,不在容器生命周期当中.这意味着即使容器删除,映射给容器的卷和卷当中的数据仍然会存在.

* 卷可以同时映射(挂载)到多个容器,多个容器共享该卷的数据.
* 卷实现了容器的迁移和移植
* 对数据卷的修改会立即在容器中生效
* 对数据卷的更新和修改,不会影响Docker镜像

![image-20200703104635678](https://img2.jesse.top/docker-volume2.png)

**Docker 卷作用**

![image-20200703104433942](https://img2.jesse.top/docker-volume1.png)

![image-20200703104851073](https://img2.jesse.top/docker-volume3.png)



---

### Docker 卷类型

Docker有两种卷类型:

1.指定绑定挂载卷

* 用户指定宿主机上的目录,映射到容器某个目录下

2.Docker自行管理的挂载卷

* 不需要指定苏追究上的目录,Docker会自行在默认的`/var/lib/docker/volumes/container_ID/`下创建一个目录映射给容器



![image-20200703105105712](https://img2.jesse.top/docker-volume4.png)

---

### 在容器中使用挂载卷

![image-20200703105412500](https://img2.jesse.top/docker-volume5.png)

---

### 实验演示

* **使用docker自行管理的挂载卷映射**

  - 将本地目录映射到容器的/data目录.使用`-v /data`命令.在挂载卷的时候,并没有指定宿主机上的路径

  ```
  [work@docker-demon ~]$ docker run -d --name b1  -v /data busybox sleep 1d
  57bb4beeed7e0eb090043c396066be204f5ed93ec031ea132aa983b9c45ad9b2
  ```

  * 通过`docker inspect b1`命令可以看到映射关系.Docker在`/var/lib/docker/volumes/Volume_ID`目录下创建了一个_data目录,并且挂载到b1容器的/data目录下

  ```
  "Mounts": [
              {
                  "Type": "volume",
                  "Name": "ca2bd3d8ac222f2eb95e63cf98a961b47aa517063386a4e4cc4431dc0b2c43c8",
                  "Source": "/var/lib/docker/volumes/ca2bd3d8ac222f2eb95e63cf98a961b47aa517063386a4e4cc4431dc0b2c43c8/_data",
                  "Destination": "/data",
  ```

  * 在宿主机的目录_data目录下新创建一个`test.txt`文件,可以看到立即同步到b1容器内的/data目录下

  ```
  #在宿主机
  [root@docker-demon ~]# touch /var/lib/docker/volumes/ca2bd3d8ac222f2eb95e63cf98a961b47aa517063386a4e4cc4431dc0b2c43c8/_data/test.txt
  
  #在b1容器内部
  [work@docker-demon ~]$ docker exec -it b1 sh
  / # ls /data
  test.txt
  / #
  ```

  > 反过来,在容器内部对该目录下进行任何数据修改,也会同步到宿主机相关目录下

  #####  容器被删除,数据还会继续保留吗?

  * 删除容器测试一下

  ```
  [work@docker-demon ~]$ docker stop b1
  b1
  [work@docker-demon ~]$ docker rm b1
  b1
  ```

  宿主机本地目录依然存在,数据并没有被随之删除

  ```
  [root@docker-demon ~]# ll /var/lib/docker/volumes/ca2bd3d8ac222f2eb95e63cf98a961b47aa517063386a4e4cc4431dc0b2c43c8/_data
  total 0
  -rw-r--r-- 1 root root 0 Jul  3 11:03 test.txt
  ```

  ##### 虽然数据依然存在,但是也可以看出来这种挂载方式非常不友好之处在于,时间一长很难找到当时映射的目录在宿主机的具体路径.特别是容器删除以后..也就是说宿主机的挂载卷很难维护

  ---

* **指定目录映射**

第二种方式是将宿主机上指定的一个目录(这个目录可以事先并不存在,Docker会自动创建)映射到容器内部

映射方式: `-v HOST_DIR:CONTAINER_DIR`

```
[work@docker-demon ~]$ docker run -d --name b1  -v/data/b1:/data busybox sleep 1d
57bb4beeed7e0eb090043c396066be204f5ed93ec031ea132aa983b9c45ad9b2
```

此时,宿主机会自动创建一个`/data/b1`的目录,并且映射给容器

```
[root@docker-demon ~]# docker inspect b1
"Mounts": [
            {
                "Type": "bind",
                "Source": "/data/b1",
                "Destination": "/data",
```

在宿主机上的`/data/b1`目录下创建一个文件,会同步到容器的`/data/`目录.反之,在容器内部创建文件,也会同步到宿主机

```
[root@docker-demon ~]# cat /data/b1/hello.txt
hello world

#容器内部会产生一个hello.txt文件
[root@docker-demon _data]# docker exec -it b1 sh
/ # ls /data
hello.txt   hello1.txt  opsdir
/ # cat /data/hello.txt
hello world
```

* **volume卷映射**

这种方法也可以映射卷,具体操作如下

1.先创建一个volume卷.命令:`docker volume creante VOLUME_NAME`

```
[root@docker-demon ~]# docker volume create jesse
```

2.然后将新创建的jesse卷映射到容器,映射方式大体一样

```
[work@docker-demon ~]$ docker run -d --name b3  -v jesse:/data busybox sleep 1d
[root@docker-demon ~]# docker inspect b3
"Mounts": [
            {
                "Type": "volume",
                "Name": "jesse",
                "Source": "/var/lib/docker/volumes/jesse/_data",
                "Destination": "/data",
                "Driver": "local",
                "Mode": "z",
                "RW": true,
                "Propagation": ""
            }
```

> 还可以通过这种方式映射: docker run -d --name b1  --mount source=jesse,targe=/data busybox sleep 1d

这种方式使用的也比较少,因为`/var/lib/docker`的docker默认存储路径一般在Linux系统的/根目录下,非数据目录.所以可能根目录的容量并不大,而且扩展相对来说不便

---

### 卷映射注意事项

以上三种卷映射方式均能实现容器数据持久化,也可以将宿主机上的同一个目录映射给多个容器,实现容器间数据共享.但是使用最多的还是第二种指定目录映射方式.具体指定宿主机上的一个目录映射给容器,而不是Docker自行管理卷.

以下是卷映射具体工作中的注意事项:

#####  1.不能映射到docker容器的相对目录.

对于宿主机来说,并不知道docker容器的当前工作目录,所以挂载到docker容器的相对目录是没有意义的.

##### 2.注意docker容器的目标路径是否已经有数据.

这里要特别注意一下.docker映射的时候尽量映射到容器的空目录下.或者确定映射后不影响容器启动.

**不同的卷挂载方式对容器已有数据的目录挂载效果不一样:**

这里使用nginx的容器来演示,nginx镜像的`/etc/nginx`目录下有`nginx.conf`等文件.

* 实验一. 宿主机上存在数据的目录挂载到容器里存在数据的目录

```
#将宿主机上的/data/b1目录映射到nginx容器的/etc/nginx目录,
#/data/b1目录存在数据:
[work@docker-demon ~]$ ll /data/b1
total 8
-rw-r--r-- 1 root root    0 Jul  3 17:00 hello1.txt
-rwxrwxrwx 1 root root   12 Jul  3 16:57 hello.txt
drwxr-xr-x 2 1000 1000 4096 Jul  3 17:22 opsdir

#映射给nginx容器
[work@docker-demon ~]$ docker run --name nginx -d -p 80:80 -v /data/b1:/etc/nginx nginx:alpine
2966b897332673a0429614428fb62638a273318d1e29dbe7bda9761002d9b4f4

#此时容器无法正常启动,因为没有找到nginx.conf配置文件
[work@docker-demon ~]$ docker logs nginx
2020/07/03 11:13:40 [emerg] 1#1: open() "/etc/nginx/nginx.conf" failed (2: No such file or directory)
nginx: [emerg] open() "/etc/nginx/nginx.conf" failed (2: No such file or directory)

#删除容器,重新映射.这次不后台运行,而是使用-it交互式界面,运行sh命令,进入sh环境.
[work@docker-demon ~]$ docker run --name nginx -it -p 80:80 -v /data/b1:/etc/nginx nginx:alpine sh
/ #

#查看/etc/nginx目录
/ # ls /etc/nginx
hello.txt   hello1.txt  opsdir
/ #
```

经过上次测试,发现指定目录挂载方式,会将宿主机上的目录数据覆盖容器下的目录的数据

* 实验二.宿主机空目录挂载到容器里存在数据的目录

```
#宿主机的/data/b2为空目录
[work@docker-demon ~]$ ls /data/b2
ls: cannot access /data/b2: No such file or directory

#映射到容器的/etc/nginx目录下,发现该数据已经被宿主机的空目录覆盖
[work@docker-demon ~]$ docker run -it --name nginx -v /data/b2:/etc/nginx nginx:alpine sh
/ # ls /etc/nginx
/ #
```

通过以上实验,发现对于指定卷映射方式来说,无论宿主机上的目录是否为空,挂载到容器的目录后,都会覆盖容器目录下的数据.

---

* 实验三.宿主机上存在数据的Volume卷挂载到容器里存在数据的目录

```
#创建一个卷.并且往卷里写入数据
[work@docker-demon ~]$ docker volume create jesse_nginx
jesse_nginx

#查看jesse_nginx卷的路径
[work@docker-demon ~]$ docker inspect jesse_nginx
[
    {
        "CreatedAt": "2020-07-03T22:36:23+08:00",
        "Driver": "local",
        "Labels": {},
        "Mountpoint": "/var/lib/docker/volumes/jesse_nginx/_data",
        "Name": "jesse_nginx",
        "Options": {},
        "Scope": "local"
    }
]

#往卷的目录/var/lib/docker/volumes/jesse_nginx/_data写入数据
[root@docker-demon ~]# sudo echo "hello world" > /var/lib/docker/volumes/jesse_nginx/_data/test.txt

#新建一个测试文件
[root@docker-demon ~]# ls /var/lib/docker/volumes/jesse_nginx/_data
test.txt
```

将该卷映射给nginx容器的/etc/nginx目录.发现容器内目录已经被卷下的数据覆盖

```
[root@docker-demon ~]# docker run -it  -v jesse_nginx:/etc/nginx nginx:alpine sh
/ # ls /etc/nginx
test.txt
/ #
```

* 实验4.将空目录的卷挂载到容器里存在数据的目录

```
#创建一个新的卷.此时卷路径为空
[root@docker-demon ~]# docker volume create jesse_nginx2
jesse_nginx2

#挂载到容器的/etc/nginx目录下.发现并没有覆盖容器内的数据
[root@docker-demon ~]# docker run -it  -v jesse_nginx2:/etc/nginx nginx:alpine sh
/ # ls /etc/nginx
conf.d          fastcgi_params  koi-win         modules         scgi_params     win-utf
fastcgi.conf    koi-utf         mime.types      nginx.conf      uwsgi_params
/ #

#查看宿主机jesse_nginx2卷下的数据,数据确实存在
[root@docker-demon ~]# ls /var/lib/docker/volumes/jesse_nginx2/_data/
conf.d  fastcgi.conf  fastcgi_params  koi-utf  koi-win  mime.types  modules  nginx.conf  scgi_params  uwsgi_params  win-utf
[root@docker-demon ~]#
```

通过以上2个实验得出.当挂载一个空目录的Volume卷到容器时,并不会覆盖容器里原本已存在的数据,但是如果是非空目录,仍然会覆盖

---

