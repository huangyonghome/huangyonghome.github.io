---
title: net.ipv4.tcp_tw_recycle踩的坑
date: 2019-01-7 09:59:58
tags: experence
categories: experence
comments: true
copyright: true
---



## net.ipv4.tcp_tw_recycle踩的坑

参考博文:https://www.cnxct.com/coping-with-the-tcp-time_wait-state-on-busy-linux-servers-in-chinese-and-dont-enable-tcp_tw_recycle/

在做系统调优的时候希望能加快`TIME_WAIT`状态的回收，通常将`net.ipv4.tcp_tw_recycle`选项开启，但是,请注意这里有个坑

### 故障现象

我们阿里云上有一台BETA测试服务器自从上次将CPU配置从4核升级到8核,然后重启服务器以后..访问该服务器上的web网站经常出现卡顿的现象,大概是每隔几分钟出现一次,很快又恢复.

1.初步怀疑是BETA服务器性能不行,包括后端数据库拥挤导致

但是经过排查发现,故障发生时,服务器性能没有任何压力,后端数据库也正常.

2.在服务器上创建一个hello world 的静态网站,发现连静态网站都无法访问.

在访问网站的时候,长时间停留在TCP_NODELAY set阶段:

<!--more-->

```
[root@localhost ~]$curl -vvv "http://test.beta.haoshiqi.net"
* About to connect() to opadmin.beta.haoshiqi.net port 80 (#0)
*   Trying 114.55.108.51...
* TCP_NODELAY set
```

通过telnet工具发现BETA服务器的80端口不通.



3. 转而怀疑是办公室网络问题.

但是办公室访问其他阿里云服务器,访问线上环境,日常办公访问互联网又一切正常,在使用mtr,traceroute等工具排查路径节点时也一切正常,没有丢包线下.ping BETA服务器也没有任何丢包现象.



4. 转而怀疑是BETA服务器问题.

但是在办公室网络访问出现故障时的同一时间,其他阿里云服务器访问BETA服务器上的网站又是一切正常,没有任何故障.用手机4G网络访问BETA也一切正常.



其他线索:

在晚上下班后,BETA服务器没有访问量时,从办公室访问BETA服务器一切正常,故障没有复现



5.经过抓包发现客户端发出了大量的TCP重传,但是服务器没有回应SYN报文.

BETA服务端tcpdump抓包命令

```
sudo tcpdump -s 0 -i eth1 host 27.115.51.166 -w /tmp/beta_server.cap 
```

客户端本地tcpdump抓包命令

```
sudo tcpdump -s 0 -i en4 host 114.55.108.51 -w ~/Desktop/beta.cap 
```

当故障出现时，服务器端其实是已经接送到TCP三次握手的SYN包，但不回应ACK给客户端，导致客户端一直处于等待状态.

----

### 原因分析

后来发现是系统内核优化时,开启了net.ipv4.tcp_tw_recycle参数.这个参数用来优化快速回收TIME_WAIT的socket.但是默认tcp_timestamps也是开启的.当开启了`tcp_tw_recycle和timestamps选项后，当连接进入`TIME_WAIT`状态后，会记录对应远端主机最后到达分节的时间戳。如果同样的主机有新的分节到达，且时间戳小于之前记录的时间戳，即视为无效，相应的数据包会被丢弃;

我们的BETA服务器在上班时间使用比较频繁.这也导致了在上班时间内一个公网IP（经过NAT）大量地去反问服务器，不同客户端的时间可能不一致，所以就会出现时间戳错乱的现象，于是后面的数据包就被丢弃了，具体的表现通常是是客户端明明发送的SYN，但服务端就是不响应ACK.

---

### 总结

很多时候我们在做系统调优的情况时，都是直接在网上看一堆的内核参数，然后直接使用，却没有仔细研究过，很多时候，坑只有是自己拆过后才能记住，像这次这个`net.ipv4.tcp_tw_recycle`故障网上很多分享，看过和自己碰到过还真不一样，好记性不如烂笔头，慢慢记录，慢慢积累，坑应该会越来越少的；

