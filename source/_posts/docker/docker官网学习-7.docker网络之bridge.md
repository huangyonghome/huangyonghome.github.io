---
title: docker学习笔记---docker网络之bridge
date: 2020-06-29 11:59:58
tags:  docker
categories: docker
comments: true
copyright: true
---



## docker官网学习--docker网络之bridge

本节介绍docker基础网络概念.以便能认识和利用各种不同的网络类型功能.

docker的网络支持插件化,驱动化定制.有一些网络驱动已经默认集成到docker中.docker网络主要有以下类型

* bridge
* host
* overlay
* macvlan
* none
* 其他网络插件

---

<!--more-->

#### bridge

​    bridge是docker默认的网络驱动.如果在`docker run`启动一个容器时没有指定任何网络驱动.那么默认就是bridge桥接网络.桥接网络通常适用于应用进程部署在多个独立的容器中,并且容器之间需要互相通信的场景中.

   在docker环境中.bridge使用软件桥接允许容器之间通过同一个bridge互联.隔离没有连到这个bridge网络的其他容器网络.docker网络自动创建iptables规则阻止其他网络的容器访问.

当Docker进程启动时，会在主机上创建一个名为docker0的虚拟网桥，此主机上启动的Docker容器会连接到这个虚拟网桥上。虚拟网桥的工作方式和物理交换机类似，这样主机上的所有容器就通过交换机连在了一个二层网络中。

从docker0子网中分配一个IP给容器使用，并设置docker0的IP地址为容器的默认网关。在主机上创建一对虚拟网卡veth pair设备，Docker将veth pair设备的一端放在新创建的容器中，并命名为eth0（容器的网卡），另一端放在主机中，以vethxxx这样类似的名字命名，并将这个网络设备加入到docker0网桥中。可以通过brctl show命令查看。

在Linux中.可以使用brctl命令查看和管理网桥(需要先安装bridge-utils软件包).例如查看本机上的网桥及其端口

```
[work@docker conf.d]$sudo brctl show
bridge name	bridge id		STP enabled	interfaces
br-17ace6d9d81a		8000.024236cdd4d7	no		veth82ed0e5
br-d9897c225d25		8000.024237d1c9f6	no		veth6981090
							veth8e29dbf
							vethd511728
docker0		8000.0242322e2e42	no		veth0a0e27d
							veth0c50104
							veth3fe7f4d					
```

docker 0网桥下关联了很多vethxxxxx规范命名的interfaces.每一个vethxxxx接口对应一个docker容器.在docker容器中一般是eth0的网卡

```
[work@docker conf.d]$docker exec -it nginx ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.8  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:acff:fe11:8  prefixlen 64  scopeid 0x20<link>
        ether 02:42:ac:11:00:08  txqueuelen 0  (Ethernet)
        RX packets 1428365  bytes 189142687 (180.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1403620  bytes 287806317 (274.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```

bridge网桥就是这样通信的,docker服务端通过vethxxxx端口和容器的eth0虚拟网卡进行通信.docker容器将宿主机的docker 0虚拟网卡的IP作为它的网关:

```
[work@docker conf.d]$docker exec -it nginx route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.17.0.1      0.0.0.0         UG    0      0        0 eth0
```



bridge模式是docker的默认网络模式，不写--net参数，就是bridge模式。使用docker run -p时，docker实际是在iptables做了DNAT规则，实现端口转发功能。可以使用iptables -t nat -vnL查看。

bridge网络模式如下所示:

![](http://cdn.img2.a-site.cn/pic.php?url=aHR0cDovL21tYml6LnFwaWMuY24vbW1iaXovUVAwQVk3dGRKblV4eFJNWjRRcDl0b21GaFFRMDNYVUViTWFab1lmbU9pYk56NDZwc0FMcDk0bHR1MllTNVZHMmZtNGUxTTNwM0tOUmVQN04xZVh2OHlBLzA/d3hfZm10PXBuZw==)

​    bridge网桥是docker的默认网络驱动.如果用户在创建容器时自定义了Bridge网络.那么自定义Bridge要优于docker默认的Bridge

 **用户定义的bridge和默认bridge的区别**

* 用户定义的bridge在多个容器之间提供更好的隔离性和协调性.

  连到同一个自定义的bridge的容器之间的所有端口互通.而无需通过-p参数暴露到宿主机.这让容器之间的通信更简单,而且提供更好的安全性.例如:

  连到同一个自定义的bridge网络的Nginx容器和mysql容器.及时mysql容器没有暴露任何端口.nginx也可以访问mysql容器的3306端口.

  而默认的Bridge网络,则需要将mysql容器通过`-p`参数暴露3306端口给宿主机.

* 自定义bridge网络提供容器的主机名DNS解析

​        默认的bridge网络下的容器间不能通过主机名互相访问,只能通过IP地址.(除非使用—link参数,但是这个参数已经废弃).而用户自定义的bridge网络则可以直接访问对方的主机名.

* 自定义bridge网络配置更方便

​        配置一个默认bridge网络,会影响到全局所有容器.而且需要重启docker服务.使用`docker network create`可以创建一个自定义bridge网络.,而且可以分别配置

---

#### 创建和管理自定义bridge网络

创建命令:

```
#创建自定义网络名称为my-net
docker network create my-net

#还可以指定网络号,子网掩码等信息
[root@localhost ~]$docker network create --help

Usage:	docker network create [OPTIONS] NETWORK

Create a network

Options:
      --attachable           Enable manual container attachment
      --aux-address map      Auxiliary IPv4 or IPv6 addresses used by Network driver (default map)
      --config-from string   The network from which copying the configuration
      --config-only          Create a configuration only network
  -d, --driver string        Driver to manage the Network (default "bridge")
      --gateway strings      IPv4 or IPv6 Gateway for the master subnet
      --ingress              Create swarm routing-mesh network
      --internal             Restrict external access to the network
      --ip-range strings     Allocate container ip from a sub-range
      --ipam-driver string   IP Address Management Driver (default "default")
      --ipam-opt map         Set IPAM driver specific options (default map)
      --ipv6                 Enable IPv6 networking
      --label list           Set metadata on a network
  -o, --opt map              Set driver specific options (default map)
      --scope string         Control the network's scope
      --subnet strings       Subnet in CIDR format that represents a network segment
```

删除自定义网络:

```
#如果有容器正在使用该网络,需要先断开容器
docker network rm my-net
```



---

#### 管理自定义网络下的容器

创建一个自定义网络下的容器

```
#在创建容器的时候指定自定义网络名

docker create --name my-nginx \
  --network my-net \
  --publish 8080:80 \
  nginx:latest
```

将一个正在运行的容器关联(移除)自定义网络

```
#关联容器和自定义网络命令格式:
docker network connect

#例如,将一个正在运行的mysql容器关联到my-net网络下
docker network connect my-net mysql

#相反从自定义网络下移除一个容器命令:
docker network disconnect

#例如,将一个正在运行的mysql容器从my-net网络下移除
docker network disconnect my-net mysql
```

---

### 实验:

* **默认的bridge网络,容器之间无法互相访问对方的主机名.只能通过iP地址通信**

```
[root@localhost ~]$docker run -itd --rm --name=busybox busybox
b3c8be3b3b716579caf11d3852f6c6e04a41b4dc020d9478be1b1f3a4d76cf1f

[root@localhost ~]$docker run -itd --rm --name=busybox1 busybox
a54d962f6a692d96f0bc2fbca37e1b47e59e6b61541f42b8d6872a5008a46b87

[root@localhost ~]$docker exec busybox ping busybox1
ping: bad address 'busybox1'
[root@localhost ~]$

#通过IP地址可以通信

[root@localhost ~]$docker exec busybox ping 172.17.0.8
PING 172.17.0.8 (172.17.0.8): 56 data bytes
64 bytes from 172.17.0.8: seq=0 ttl=64 time=0.136 ms
64 bytes from 172.17.0.8: seq=1 ttl=64 time=0.079 ms
64 bytes from 172.17.0.8: seq=2 ttl=64 time=0.131 ms
```

* **自定义的Bridge网络,可以在容器之间互相访问主机名**

```
#创建一个自定义网络
[root@localhost ~]$docker network create jesse
e10a936177681cbfc321f67f961f2a717079ef1790c50e82381296fa77bd7d5f

[root@localhost ~]$docker network ls | grep jesse
e10a93617768        jesse                  bridge              local

#创建容器,使用network参数指定网络
[root@localhost ~]$docker run --name busybox1 -itd --network jesse --rm busybox
aedf164ea1769741ae6480583abdc838022f49506761d4054daabdb7fffcd852

[root@localhost ~]$docker run --name busybox2 -itd --network jesse --rm busybox
523bdd54fa12423e25a8ac84f9faef1679ee6031f2d5d2a87c0cacd64f8650ad

[root@localhost ~]$docker exec busybox1 ping busybox2
PING busybox2 (172.20.0.3): 56 data bytes
64 bytes from 172.20.0.3: seq=0 ttl=64 time=0.116 ms
64 bytes from 172.20.0.3: seq=1 ttl=64 time=0.078 ms

```

* 连接到不同的bridge网络下的容器互相之间网络隔离

```
docker network create bridge
docker network create kong-net

[work@docker conf.d]$docker run --name busybox1 --network bridge -itd busybox
f95a229aa7eb5d7022bef4441a075b0ad37ecc50e2a02015f09790d23b28dc33

[work@docker conf.d]$docker run --name busybox2 --network kong-net -itd busybox
a9236a25199cc43a899285462afa851a52cdaf871776a93829897676fc7dd82c

#busybox1和busybox2不在同一个IP网段
#busybox1的IP
[work@docker conf.d]$docker inspect busybox1 
172.17.0.11

#busybox2的IP
[work@docker conf.d]$docker exec -it busybox2 ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 02:42:AC:12:00:05
          inet addr:172.18.0.5  Bcast:0.0.0.0  Mask:255.255.0.0
          
#主机名无法访问
[work@docker conf.d]$docker exec -it busybox1 ping busybox2
ping: bad address 'busybox2'

#busybox1容器也无法ping busybox2容器的IP
[work@docker conf.d]$docker exec -it busybox1 ping 172.18.0.5
PING 172.18.0.5 (172.18.0.5): 56 data bytes
^C
--- 172.18.0.5 ping statistics ---
32 packets transmitted, 0 packets received, 100% packet loss
```

> 由于博客不能识别go模板语法,所以省去了go模板,直接用docker inspect busybox1命令来代替.实际场景中该命令无法直接获取容器IP

#### 将容器从自定义bridge网络中移除

```
#将jesse从jesse网络移除
[root@localhost ~]$docker network disconnect jesse busybox1

#此时jesse网络下只有busybox2容器
[root@localhost ~]$docker network inspect jesse

#此时busybox1容器的网卡被移除了
[root@localhost ~]$docker inspect busybox1 
<no value>

root@localhost ~]$docker exec -it busybox1 ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:11 errors:0 dropped:0 overruns:0 frame:0
          TX packets:11 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:618 (618.0 B)  TX bytes:618 (618.0 B)
```

#### 将一个正在运行的容器加入到自定义bridge网络

```
#将jesse加回到jesse网络
[root@localhost ~]$docker network connect jesse busybox1

#获得新的IP地址
[root@localhost ~]$docker inspect busybox1 
172.20.0.2
```

---

#### 宿主机转发容器端口

默认情况下,bridge网络不会转发外部的请求到容器.开启转发需要更改2个设置:

1.修改内容,开启转发

```
sysctl net.ipv4.conf.all.forwarding=1

sysctl -p
```

2.更改iptables的转发的默认规则

```
sudo iptables -P FORWARD ACCEPT
```



#### 更改默认Bridge网络配置

docker0网桥是在docker daemon启动时自动创建的.IP默认为172.17.0.1/16.所有连到docker 0网桥的docker容器都会在这个IP范围内选取一个未占用的IP使用.并连接到docker 0网桥上

docker提供了一些参数帮助用户自定义docker0网桥的设置

* —bip=CIDR: 设置Docker0的IP地址和子网范围.使用CIDR格式.例如192.168.0.1/24.需要注意的是这个参数仅仅是配置docker0的,对其他自定义的网桥无效.
* —fixed-cidr=CIDR:限制docker容器获取iP的范围.默认情况下docker容器获取的IP范围为整个docker0网桥的IP地址段,也就是—bip指定的地址范围.此参数可以将docker容器缩小到某个子网范围.
* —mtu=BYTES: 指定docker0的最大传输单元(MTU)

```
#更改daemon.json配置文件.下面这个例子修改了docker0网络的网段地址
vim /etc/docker/daemon.json

{
  "registry-mirrors": ["https://registry.docker-cn.com"],
  "bip": "192.168.1.5/24",
  "mtu": 1500,
  "dns": ["114.114.114.114","114.114.115.115"]

}

#重启docker服务
[root@localhost ~]$systemctl restart docker

[root@localhost ~]$ifconfig | grep -A 5 docker0
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.5  netmask 255.255.255.0  broadcast 192.168.1.255
        ether 02:42:89:26:d1:c9  txqueuelen 0  (Ethernet)
        RX packets 1072032  bytes 61915874 (59.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1997027  bytes 1934238839 (1.8 GiB)
```

---

