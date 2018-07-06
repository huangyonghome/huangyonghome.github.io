---
title: BGP  Concept
date: 2018-06-12 22:59:58
tags: [network,Cisco,route ]
categories: [Network,Route,BGP ]
comments: true
copyright: true
---



## BGP Concept

### 1.概念

BGP（border gateway protocol）边界网关路由协议。

**内部网关路由协议（IGP)**：这种协议用在自治系统内部，比如RIP,EIGRP,OSPF,ISIS，这些都术语IGP。IGP并不是一种路由协议，而是内部网关路由协议的统称。

**外部网关路由协议(EGP)**:BGP属于外部网关路由协议。

**自治系统**是个16位的数字，取值范围是1-65535.其中64512-65535保留给私用，类似于私有地址。

**BGP**是路径矢量路由协议，不要求采用像OSPF和ISIS那样的层次化网络设计。根据路径矢量属性来选择最优路径。

**BGP**是唯一一种使用**TCP**作为传输层协议的IP路由协议。端口号是179.OSPF和EIGRP直接运行在IP层之上，IS-IS位于网络层，RIP使用UDP传输。因为TCP是一对一的传输协议，所以BGP不使用广播或者组播更新，建立邻居也是neighbor指定的邻居路由器，而不是发送组播hello包与所有邻居路由器建立邻接关系。

**BGP**没有HELLO包，取而代之的是发送KEEPALIVE消息，没有周期性更新，只有触发更新。

内部BGP（IBGP）**管理距离**是200.外部BGP(EBGP)管理距离是20

和IGP一样，BGP也是采用下一跳的逐跳路由协议，只不过IGP路由协议是以**下一台路由器**作为下一跳，但是BGP的下一跳是以**AS号自治系统**为单位，一个自治系统内部可能包含非常多的路由器。BGP下一跳的地址是**下一个AS号的入口接口IP**。也就邻居路由器的IP.

BGP路由有一个路径列表，记录了这条路由经过的所有AS号，BGP路由器不会接受路径列表中包含其AS号的路由选择更新，这种机制也被称为**BGP的水平分割**原则，用来防环

---

<!--more-->

### 2.GBP表

1.邻居表 

2.GBP表 

3.IP路由表 

运行BGP协议的路由器有一张BGP表，或者叫BGP拓扑表，拓扑数据库表。它独立于IP路由表。从BGP表中选出最优的路径放入路由表summer。但即便BGP表中的路由不是最优没有放入路由表，但该路由仍然会被通告的其他BGP邻居。 

---

### 3.GBP邻居

EBGP之间一般有物理链路直连，两个路由器之间在同一个子网建立邻居关系。IBGP内部建立邻居没必要有直连的物理链路，邻居之间也可以不在同一个子网。如果两台路由器不在同一个子网建立邻居，需要确保路由器之间的路由可达，

IBGP一般实行全互联连接，且同时运行一种IGP协议。例如：OSPF,EIGRP。

IBGP之间一般通过环回接口建立邻居，如果EBGP之间有多条物理链路连接（为实现负载和冗余）也通过环回接口建立邻居。这样做的好处是：如果两台路由器之间某条物理链路down了，仍然有其他的物理链路可达对方。

##### *BGP邻居状态：

1.空闲（idle）：路由器搜索路由表，看看是否有前往邻居的路由

2.连接（connect）路由器找到了前往邻居的路由，并且完成了TCP三次握手

3.打开发送（open sent）：发送一条打开消息，其中包含BGP会话参数。

4.打开确认（open confirm）路由器就建立会话的参数达成一致

5.active

6.已建立（established）:邻居关系已经建立

---

### 4.GBP属性 

BGP使用属性来确定前往网络的最佳路径。属性可以是：

##### 1.公认的（well-known）:所有BGP路由器都能识别的，并且被传输给所有BGP邻居。分为：

* 强制的（mandatory）           #公认强制的
* 自由决定的（discretionary）#公认自由决定的

##### 2.可选的（optional）：非公认属性，分为：

* 传递的（transitive）
* 非传递的（nontransitive）

##### 3.部分的（partial）

 

#### 公认强制属性：

* AS路径（AS-path）;
* 下一跳（next-hop）
* 源头（origin）.
* 公认自由决定属性：
* 本地优先级（local preference）
* 原子聚合（Atomic aggregate） 

#### 可选传递属性：

* 聚合站（aggregator）
* 共同体（community） 

#### 可选非传递属性：

* 多出口鉴别器（multi-exit-discriminator.MED）

 

>  除外还有权重属性（weight）本地有效，不会传播给其他BGP路由器

---

#### 5.GBP消息类型

**1.Open packet** 

Include hole time (30s) .and BGP router ID

**2.Keepalive packet **

**3.Update **

Information for routes ,path , 

Includes path attributes and networks 

**4.Notification**

when error is detected

---

#### 6.network 宣告 

BGP的network 命令和以往任何路由协议都不同。 BGP的network命令不是通告某个接口参加BGP进程，而是宣告路由表中的所有路由出去，可以是直连的，静态的，或者从IGP中学到的。network命令不是为了建立邻居，而是把自己的路由通告给对方。

network特点：

* 必须精确宣告网络号和子网掩码 

格式：Network X.X.X.X mask X.X.X.X

* 宣告的路由必须存在于自己的路由表中，否则宣告失败。

---

#### 7.BGP同步规则 

同步规则：当一个路由器通过IBGP学到一条BGP路由时，默认不会使用，也不通告给外部AS的邻居，除非从IGP也学到这条路由。 

例如:

![BGP] {% qnimg network/BGP.png %}

当R3通过IBGP学到位于R4的172.16.1.0这条路由的时候，默认不会通告给位于外部AS内的R5，除非是通过本地AS内的IGP（EIGRP）路由协议也学到172.16.1.0这条路由。 

此举是为了防止路由黑洞，现在的IOS默认情况下关闭了同步功能 



开启命令：

```
router bgp 123
synchronization 
```

---



#### 8.BGP认证

BGP认证非常简单，只需要在指定邻居时加上password 参数： 

```
R1(config)#router bgp 123
R1(config-router)#neighbor 2.2.2.2 pass
R1(config-router)#neighbor 2.2.2.2 password ?
  <0-7>  Encryption type (0 to disable encryption, 7 for proprietary)
  LINE   The password
```

---

#### 9.BGP会话重置 

BGP是触发更新，所以当我们需要发送更新的时候需要将BGP会话重置来发送一个触发更新，有3个重置的办法：

**1.硬重置**

在特权模式下输入：clear ip bgp * 

>  默认情况下回重置了所有BGP的信息，包括邻居关系会down掉 

如果只针对某个邻居重置，则加上邻居地址,例如在R2执行下面命令： 

```
R2#clear ip bgp 1.1.1.1
R2#
*Apr 10 14:05:28.295: %BGP-5-ADJCHANGE: neighbor 1.1.1.1 Down User reset
*Apr 10 14:05:28.299: %BGP_SESSION-5-ADJCHANGE: neighbor 1.1.1.1 IPv4 Unicast topology base removed from session  User reset
R2#
*Apr 10 14:05:29.539: %BGP-5-ADJCHANGE: neighbor 1.1.1.1 Up
```



**2.软重置**

在特权模式下输入：clear ip bgp *  soft

软重置不会重置邻居关系，会重新发送或者接受路由更新 

在后面还有2个参数，分别代表发送和接受路由的重置 

```
 R2#clear ip bgp * soft ?
  in    Soft reconfig inbound updates
  out   Soft reconfig outbound updates
  slow  Forcefully clear slow-peer status and move it to original update group
  <cr>
```

**3.路由刷新：动态的入站软重置**
命令：clear ip bgp * soft in。或者 clear ip bgp * in


