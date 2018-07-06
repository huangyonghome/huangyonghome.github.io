---
title: HSRP实验
date: 2018-06-16 07:59:58
tags: [network,Cisco ]
categories: [Network,Switch ]
comments: true
copyright: true
---



## HSRP实验



### 1.实验拓扑：

{% qnimg network/hsrp.png %}

---

<!--more-->

### 2.实验环境

主机属于VLAN1，IP地址：10.1.1.1  

2960为一台2层交换机，上行两条TRUNK链路连接2台三层交换机

3560三层交换机通过TRUNK口相连，上行链路是3层端口连接一台路由器。并且写入两条静态路由，一条是到达5.5.5.5，另一条是到达对方三层交换机和路由器的链路。

路由器开启一个环回接口5.5.5.5充当外网。

---



### 3实验目的：

两台三层交换机开启HSRP，观察HSRP工作过程

---



### 4.配置步骤：



* 在3560Switch1上:

```
Switch(config)#int vlan 1                    #进入VLAN接口模式

Switch(config-if)#standby 1 ip 10.1.1.254    #1是HSRP的组号，参与同一个HSRP进程的所有路由器，组号必须相同。这里是对VLAN 1做的，所以组号设置为1.。后面是一个虚拟IP地址，
```

**以下是一些可选设置：**

```
1.设置优先级。默认优先级为100.优先级越高，路由器越可能成为ACTIVE

Switch(config-if)#standby 1 priority ?

<0-255> Priority value

Switch(config-if)#standby 1 priority

2.设置HELLO时间，以及HELLO间隔。默认是3秒，间隔10秒

Switch(config-if)#standby 1 timers ?

<1-254> Hello interval in seconds

Switch(config-if)#standby 1 timers

3.设置抢占特性。默认情况下，HSRP不抢占。开启preempt参数后，优先级路由器高的可以抢占ACTIVE角色

Switch(config-if)#standby 1 preempt

```



Switch1经过speak,standby角色后成为了active网关，实际总共经过listen，speak,standby,active角色。

{% qnimg network/hsrp1.png %}

---



* 在另外一台3560上配置：

```
Switch(config)#int vlan 1

Switch(config-if)#standby 1 ip 10.1.1.254

%HSRP-6-STATECHANGE: Vlan1 Grp 1 state Speak -> Standby

```



配置基本完成。show standby和show standby brief 查看HSRP信息:



{% qnimg network/hsrp2.png %}



> 通过show standby可以看到HSRP组号，虚拟IP网关，虚拟MAC地址。Hello时间和间隔。抢占特性是否开启。活跃网关路由器，备份网关路由器，优先级等



{% qnimg network/hsrp3.png %}



> 通过show standby brief 可以更加直观的看到组号，优先级，角色状态，路由器角色，网关IP等信息

---



#### 5.测试主机访问外部网络：



* 主机配置虚拟IP地址为默认网关：

{% qnimg network/hsrp4.png %}



* 测试访问外部网络的路径

![hsrp] {% qnimg network/hsrp5.png %}



实验显示R1可以正常通过默认网关访问外部网络，并且经过的是10.1.1.2这台活跃路由器



* 查看主机的ARP记录表

{% qnimg network/hsrp6.png %}



实验显示主机是通过虚拟IP和虚拟MAC地址通信。

------



#### 6.测试抢占特性



##### 1.开启抢占特性

```
Switch(config)#int vlan 1
Switch(config-if)#standby 1 preempt
```

##### 2.把Standby交换机的优先级设置成150 

```
Switch(config-if)#standby 1 priority ?
<0-255> Priority value
Switch(config-if)#standby 1 priority 150
```

##### 3.观察HSRP的角色状态变化

交换机角色马上从standby转变成active

{% qnimg network/hsrp7.png %}



显示本交换机成为了active，对端成为了standby，并且本机优先级变成了150

{% qnimg network/hsrp8.png %}

------

#### 7.当active路由器出现问题时，查看主机所受的影响



##### 1.在主机连续ping 远端路由器 5.5.5.5 1000个数据包



##### 2.关闭ACTIVE交换机的VLAN1端口

实验显示只丢了一个包

{% qnimg network/hsrp9.png %}

------



### 第二部分 多组HSRP配置

HSRP是一个网关冗余协议，这就意味着默认情况下网络中只能有一台在工作，虽然达到了冗余的目的，但是如果所有流量都全部经由同一台网关，而另外一台网关设备完全不工作，这就造成了一种资源的浪费，以及可能带来网络中流量的拥挤。

通过配置多个HSRP组，为每组流量设置一个不同的ACTIVE网关，

---



#### 1.实验拓扑

{% qnimg network/hsrp10.png %}

---



#### 2.环境说明：

PC0和PC1分别模拟两个VLAN下的主机.

上面的三层3560交换机1作为Vlan1的根网桥以及active网关，虚拟网关IP是10.1.1.254

下面的三层3560交换机0作为Vlan2的根网桥以及active网关，虚拟网关IP是10.1.2.254

>  此次配置省略了，Vlan,trunk,vlan接口，生成树的配置

---



#### 3.配置步骤:



##### 1.配置VLAN1的HSRP

在Switch1上的Vlan1接口下配置HSRP的VLAN1组，此台设备的优先级为150，设置虚拟IP,开启抢占特性

{% qnimg network/hsrp11.png %}



##### 2.在Switch0的Vlan1接口下配置HSRP的vlan1组，此台设备的优先级保留默认设置，设置虚拟IP,开启抢占特性

```
Switch(config)#int vlan 1
Switch(config-if)#standby 1 ip 10.1.1.254
Switch(config-if)#standby 1 preempt

%HSRP-6-STATECHANGE: Vlan1 Grp 1 state Speak -> Standby
```

##### 3.配置VLAN的HSRP

* 在Switch1上的Vlan2接口下配置HSRP的VLAN2组，此台设备的优先级保留默认设置 ，设置虚拟IP,开启抢占特性

```
Switch(config-if)#int vlan 2
Switch(config-if)#standby 2 ip 10.1.2.254
Switch(config-if)#standby 2 preempt
```

* 在Switch0的Vlan2接口下配置HSRP的vlan2组，此台设备的优先级为150 ，设置虚拟IP,开启抢占特性

```
Switch(config)#int vlan 2
Switch(config-if)#standby 2 ip 10.1.2.254
Switch(config-if)#standby 2 priority 150
Switch(config-if)#standby 2 preempt
```

##### 4.检查配置

在Switch0上检查配置发现本台设备是VLAN1组的standby网关，Vlan2组的active网关

 {% qnimg network/hsrp12.png %}

 {% qnimg network/hsrp13.png %}

---



##### 5.查看每台PC的流量路径

在Vlan1的PC0上tracert 位于路由器的5.5.5.5地址。发现流量先经过10.1.1.2的网关，而10.1.1.2正是Vlan1的ACTIVE网关的真实IP

![hsrp] {% qnimg network/hsrp14.png %}

在Vlan2的PC1上tracert位于路由器的5.5.5.5地址：发现流量先经过10.1.2.3的网关，而10.1.2.3正是Vlan2的ACTIVE网关的真实IP

 {% qnimg network/hsrp15.png %}

------



### 第三部分: HSRP 的Track 功能配置

HSRP的track功能可以追踪上行链路的情况，如果配置了track，一旦上行链路失效，则默认情况下HSRP的路由器降低10个优先级。



#### 1.配置命令：

{% qnimg network/hsrp16.png %}

真实的设备中，接口后面可以指定一旦上行链路失效，会降低多少优先级。但是模拟器中没有这个功能，默认是降低10个优先级

把上面的Switch 1的上行接口F0/1 shutdown 之后，可以看到Switch的优先级变化：优先级从150降低到140

{% qnimg network/hsrp17.png %}



> Tips:真实的设备中，还可以track路由，既一条指定的路由失效后，HSRP的优先级会降低。