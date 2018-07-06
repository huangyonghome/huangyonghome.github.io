---
title: 02-BGP Lab
date: 2018-06-16 08:59:58
tags: [network,Cisco,route ]
categories: [Network,Route,BGP ]
comments: true
copyright: true
---



## BGP LAB



### 1.实验拓扑

配置三个AS，运行BGP协议。 

 {% qnimg network/bgp1.png %}

---

<!--more-->


### 2.BGP配置

R1配置： 

  {% qnimg network/bgp2.png %}

no synchronization：同步默认被关闭

no auto-summary：汇总默认被关闭

bgp router-id 1.1.1.1：手工指定R1路由器的ID。注意前面要加参数“BGP”

network 1.1.1.0 mask 255.255.255.0：通告自己的1.1.1.0的路由。

> 注意：在BGP中network命令代表的意思和IGP完全不一样，在IGP里network表示在某个接口启用IGP协议，并和邻居路由器建立邻接关系，发送和接受路由更新。在BGP里，不用把接口宣告进BGP，network表示把自己路由表里的路由宣告给邻居路由器。并且必须先建立邻居关系才能宣告路由，如果邻居无法建立则network命令没有任何意义。如果network手工宣告一条路由，但是该路由不在路由表中，BGP不宣告该路由出去。在这里R1宣告自己环回口的路由）

neighbor 12.1.1.2 remote-as 65100：指定邻居，并且指定邻居所在的AS号。



R2配置： 

  {% qnimg network/bgp3.png %}



利用环回口和R3.R4.R5建立邻居。即便没有和R5直接相连。同时和R1建立EBGP邻居关系。



R3配置： 

  {% qnimg network/bgp4.png %}



R4配置： 

  {% qnimg network/bgp5.png %}



R5配置： 

  {% qnimg network/bgp6.png %}



R6配置： 

  {% qnimg network/bgp7.png %}

---



### 3.检查配置

##### 1.查看R1的BGP表：show  ip bgp 

  {% qnimg network/bgp8.png %}



*：星号代表路由可达。

\>：代表这是条最优路由。

network:路由前缀，网络号。

next hop:0.0.0.0 表示是本台路由器通告的路由。

metric：这就是MED属性值，默认是0

locprf:本地性能值，默认为100.

weight:权重属性值，本地通告的路由属性值默认是32768.其他路由器通告的默认值是0

path：该路由通告到本地时经过的AS号，在这里表示是从65100自治系统学来的路由。

i:表示始发路由器是通过network命令，将该路由通告到BGP中。



##### 2.查看R1的路由表 show ip route：将BGP表中最优路由放入路由表 

  {% qnimg network/bgp9.png %}



##### 3.查看R2的BGP表：show  ip bgp

  {% qnimg network/bgp10.png %}



> 在R2上我们并没有看到R3,R4.R5的信息。 



##### 4.查看R3的BGP表：show  ip bgp

  {% qnimg network/bgp11.png %}

> 在这只能看到自己本台路由器通告的路由，看不到任何邻居 



##### 5.去R2看看邻居表：show ip bgp neighbors.以下是部分输出： 

  {% qnimg network/bgp12.png %}



从这里看到和R1的邻居关系状态是established.表示邻居已经成功建立。

BGP的版本是4.远端AS号是65000，外部链路。邻居路由器的IP地址和路由器ID。

keepalive间隔60秒，Hold time是三倍间隔，

显示发送和接收了open,notifications,updates,keepalives,route refresh的数据包。



  {% qnimg network/bgp13.png %}



在这里看到邻居3.3.3.3的状态是active。内部链路，AS号是65100.或许不到对方的路由器ID。并且没有任何报文收发。

造成BGP邻居之间active状态的原因主要有：

* 配置邻居地址错误，比如邻居的地址是1.1.1.1.配置成了1.1.1.2.
* update-source源地址没有指定
* AS号不对
* 没有到达对方的路由



4.4.4.4邻居也是如此。 

  {% qnimg network/bgp14.png %}



##### 6.在R2上show ip bgp summary 

  {% qnimg network/bgp15.png %}

可以看到只有R1的邻居有收发报文，其他邻居的状态是ACTIVE。 



##### 7.在R2和R3上debug ip bgp: 

  {% qnimg network/bgp16.png %}

显示OPEN消息被拒绝，没有路由。

 这是因为环回口之间没有路由，邻居不可达。所以在IBGP内部要配置IGP协议。这里配置EIGRP协议。



##### 8.在R2上配置EIGRP 

  {% qnimg network/bgp17.png %}

>  在这，我仅仅列出了R2的配置，R3.R4.R5也做了同样的配置。需要注意的是R2连接R1的接口不需要运行EIGRP协议。 



查看R2的路由表： 

  {% qnimg network/bgp18.png %}

再次查看R2的BGP表： 

  {% qnimg network/bgp19.png %}



邻居仍然还未建立。

debug ip bgp:

  {% qnimg network/bgp20.png %}

显示open消息active状态，连接远程主机失败，这是因为R2查看路由表，发现去往邻居的源地址是23.1.1.2和24.1.1.2。。但是在邻居路由器上指定的R2路由器邻居是2.2.2.2.所以邻居路由器不识别R2的源地址.

我们需要修改R2的源地址.使邻居的地址匹配



在R2上配置:

  {% qnimg network/bgp21.png %}

在指定邻居的时候说明自己的源地址是环回0口，如果不指定源地址，路由器会查找路由表然后按路由表到对方目的地的物理接口作为出口也就是源地址。

同样，需要在R3.R4.R5路由器上做同样的配置，这里仅列入了R3的配置:

  {% qnimg network/bgp22.png %}

R3指定R2的地址是2.2.2.2。将自己的3.3.3.3作为源地址

R2指定R3的地址是3.3.3.3。将自己的2.2.2.2作为源地址

这样R2和R3之间已经建立了邻居。



再次查看R2的BGP表:

  {% qnimg network/bgp23.png %}

至此,所有邻居已经建立.

注意到去往R6路由器的环回口的路由并不是最优，而且下一跳是R6路由器的物理接口。这表明默认情况下EGP路由传递的时候下一跳始终是本台路由器的接口.



在R3上看BGP表 

  {% qnimg network/bgp24.png %}

同样，去往R1和R6的路由并不是最优，而且下一跳是路由通告者的接口 

查看R3路由表： 

  {% qnimg network/bgp25.png %}

发现没有1.1.1.0/24和6.6.6.0/24这两条路由。这说明不是最优的路由不会放进路由表。

查看R1的BGP表：

  {% qnimg network/bgp26.png %}

发现没有去往R6的路由.

这说明：不是最优的路由，不放进路由表，而且不通过给其他GBP路由器，这导致R6和R1之间学不到彼此的路由。 

##### 9.最优路由

BGP最优路由的条件：

1.同步:新版本IOS里已经自动禁用了同步。在全互联的IBGP网络里同步需要要关闭

2.下一跳.

查看R3的BGP表 

  {% qnimg network/bgp27.png %}

因为去往1.1.1.0/24的下一跳是12.1.1.1.是R1的物理接口，而R1是位于其他自治系统，所以没有运行IGP协议，所以R3没有到12.1.1.1的路由，也就是去往1.1.1.0/24不可达。同理6.6.6.0/24网络也是如此。

 同理，在R2上,去往R6路由器的下一跳也不可达

  {% qnimg network/bgp28.png %}

所以，必须要改变下一跳地址：

设置下一跳地址的最好办法是使用：next-hop-self命令。此命令经常被用于边界路由器上，用于向IBGP邻居说明去往另一个自治系统的路由下一跳指向自己本台路由器。

在R2上配置:

  {% qnimg network/bgp29.png %}



在R5上配置： 

  {% qnimg network/bgp30.png %}



查看R2的BGP表和路由表.去往R6的路由已经变成最优，且放进了路由表。

 

  {% qnimg network/bgp31.png %}

  {% qnimg network/bgp32.png %}



查看R3的BGP表 

  {% qnimg network/bgp33.png %}

发现去往R1的路由下一跳已经变成了R2。去往R6的路由下一跳变成了R5。

下一跳地址是R2和R6配置邻居时指定的update-source的源地址，而不是物理接口地址。

并且他们都是最优路由.



查看R1,R2的BGP表

  {% qnimg network/bgp34.png %}



locprf是本地性能值，默认为100.该值仅在IBGP内部有效。

前面的i表示该路由是通过IBGP邻居学到的，没有打i表示改路由是通过EBGP外部学来的。

前面的r表示该BGP学来的路由没有进入路由表，在rib-failure列表里。查看rib-failure：

  {% qnimg network/bgp35.png %}



显示3条BGP路由没有进入路由表，原因是相比IGP协议，BGP路由有更高的管理距离，所以没有进入路由表。

查看R2的路由表，发现通过EIGRP学到的路由被放进了路由表，因为管理距离更小（90）,而IGBP的管理距离是200.

  {% qnimg network/bgp36.png %}

---



### 第二部分: BGP路由器通告的更新



1.查看路由器通告的更新 

命令：

```
show ip bgp neighbor XX.XX.XX.XX advertised-routes 
```



2.查看路由器接受的路由更新 

命令:

```
show ip bgp neighbor XX.XX.XX.XX received-routes 
```

---



### 第三部分:BGP聚合

BGP通告一条汇总路由的行为就叫聚合，在IGP里称为汇总。 

#### 实验：

在R1路由器上配置4个环回口地址（200.192.16.0/22），然后通告一条汇总路由到其他路由器.通告汇总路由方法



##### 1.利用静态路由和network通告 

在IGP路由协议里，路由器通告汇总路由的时候会在本台路由上产生一条指向空接口的汇总路由。同理，在BGP里也可以手工配置一条指向空接口静态路由，然后利用Network命令宣告该汇总路由

如果不手工指定的话，本台路由表则只有明细路由，没有汇总路由，那么Network命令不会生效

* 配置方法
  * 1.手工配置一条静态路由,指向Null0
  * 2.network宣告该路由



##### 2.聚合通告 

使用aggregate-address命令 :

```
aggregate-address xx.xx.xx.xx 子网掩码
```



> Tip: 使用summary-only参数屏蔽明细路由,只显示汇总路由.在命令后面加上summary-only参数

> Note:如果明细路由通告者和路由汇总者不在一台路由器的情况。 在路由汇总的时候需要加上as-set参数







