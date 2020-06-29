---
title: DNS介绍
date: 2018-08-14 09:59:58
tags: DNS
categories: [Linux-Service ]
comments: true
copyright: true
---


### DNS介绍

IP地址虽然采用了十分记数法.但是对于人脑来说还是非常难以记忆,但是访问互联网网站又一定需要IP,为了应对这个问题.早期的时候是通过hosts文件来绑定主机名和IP的对应关系,

但是这种方法存在非常多的不足,特别是IP和主机名的对应关系越来越多的时候.hosts档案完全无法满足人们的需求.这个时候,伯克利大学发展出一套主机名IP对应系统.称为Berkeley Internet Name Domain, BIND  .也就是目前全世界使用最广泛的Domain Name System, DNS 

<!--more-->

---

DNS系统采用树状的阶层是架构,

最顶上是root.也称为根域名,

第二层是顶级域名.例如com,org,edu,gov,net,还有国家域名,比如cn(中国),jp(日本),uk(英国),us(美国)等.

和DNS相关的另外一个概念是FQDN( Fully Qualified Domain Name ).完全合格域名. FQDN由两部分组成:hostname和domain name (也就是主机名+域名).最著名的主机名是www.它是所有网站站点的主机名.

> 举个例子: www.doweidu.com  这个就是一个FQDN完全合格域名.其中doweidu.com是域名.www是主机名

---

#### DNS的查询过程

 DNS 是以类似『树状目录』的型态来进行主机名的管理的！所以每一部 DNS 服务器都仅管理自己的下一层主机 .下图演示了www.ksu.edu.tw这个主机名的DNS查询过程:

![](https://img1.jesse.top/dns.png)



1.首先，当你在浏览器的网址列输入 [http://www.ksu.edu.tw](http://www.ksu.edu.tw/) 时，你的计算机就会查询本机的hosts文件,是否绑定此主机名和IP的对应关系.

2.浏览器会查看本身的DNS缓存记录

3.客户端(也就是你本机)会向DNS服务器查询[http://www.ksu.edu.tw](http://www.ksu.edu.tw/)的IP地址

4.如果DNS服务器没有此主机的IP地址,则想最顶层的root服务器查询.但是由于 . 只记录了 .tw 的信息,并不知道具体tw下的某个主机的IP地址.此时 . 会告诉DNS服务器它不知道主机的IP ，不过它知道tw服务器在哪里，你应该向 .tw 去询问才对，然后返回tw服务器的IP地址给DNS服务器

5.DNS Server接着又到 .tw 去查询，而该部机器管理的又仅有 .edu.tw, .com.tw, gov.tw... 那几部主机，经过比对后发现我们要的是 .edu.tw 的网域，所以这个时候 .tw 又告诉 DNS server 说：你要去管理 .edu.tw 这个网域的主机那里查询，我有他的 IP ！

6.DNS Server接着又到.edu.tw去查询, .edu.tw 也没有具体主机的IP ，它会返回DNS服务器关于.ksu.edu.tw的地址，让DNS服务器去向.ksu.edu.tw服务器查询

7.当DNS server向.ksu.edu.tw服务器查询www.ksu.edu.tw主机的IP地址时,由于.ksu.edu.tw服务器管理此主机的IP地址关系,所以向DNS server返回正确的www.ksu.edu.tw主机的IP地址.

8.当DNS server查询到了正确的IP地址后,会将此主机名和IP地址缓存在自己的内存中,方便后续其他主机的查询.然后将IP地址返回给客户端,(当然,这个缓存值有时间限制,一般是24小时内,该记录就会被释放).

9.此时客户端浏览器拿到了正确的IP地址,就可以通过www.ksu.edu.tw这个域名访问远程主机的资源.如果DNS server到最后也没有查询到www.ksu.edu.tw主机的IP地址,DNS SERVER会告诉客户端此主机名不存在.客户端浏览器会返回错误信息,告诉用户此网站名不存在.



下面的dig命令详细论证了DNS的查询过程:

```
[root@localhost ~]$dig +trace www.ksu.edu.tw

; <<>> DiG 9.9.4-RedHat-9.9.4-61.el7 <<>> +trace www.ksu.edu.tw
;; global options: +cmd
.			275227	IN	NS	f.root-servers.net.
.			275227	IN	NS	g.root-servers.net.
.			275227	IN	NS	k.root-servers.net.
.			275227	IN	NS	e.root-servers.net.
.			275227	IN	NS	c.root-servers.net.
.			275227	IN	NS	j.root-servers.net.
.			275227	IN	NS	d.root-servers.net.
.			275227	IN	NS	h.root-servers.net.
.			275227	IN	NS	l.root-servers.net.
.			275227	IN	NS	b.root-servers.net.
.			275227	IN	NS	m.root-servers.net.
.			275227	IN	NS	a.root-servers.net.
.			275227	IN	NS	i.root-servers.net.
;; Received 239 bytes from 114.114.114.114#53(114.114.114.114) in 253 ms

tw.			172800	IN	NS	g.dns.tw.
tw.			172800	IN	NS	f.dns.tw.
tw.			172800	IN	NS	ns.twnic.net.
tw.			172800	IN	NS	d.dns.tw.
tw.			172800	IN	NS	a.dns.tw.
tw.			172800	IN	NS	b.dns.tw.
tw.			172800	IN	NS	i.dns.tw.
tw.			172800	IN	NS	c.dns.tw.
tw.			172800	IN	NS	e.dns.tw.
tw.			172800	IN	NS	h.dns.tw.
tw.			172800	IN	NS	anytld.apnic.net.
tw.			86400	IN	DS	40792 8 2 A05DB4B0DEB971031361BB621E8BB1B8D7346665A3D1B06EC1431ADB 7D015EE9
tw.			86400	IN	RRSIG	DS 8 1 86400 20180822170000 20180809160000 41656 . ORWOjWoiZqcRkVip1DvoUVvxHIxQZJBsT93obIYJEdw61Vuam6IhNmYV u+cNQF9HqsfiVSJaiekiK2ERjpmkaNdJfeK/zAkYhKLa6l0JzZoJVxjj Ul/HKZIvbB07MjyZP1lM7HCAlHSAR1C+g9H1owUgW031G4L+cIQ7PVaa bBFs/C8JJbyOqR4RwkCPWmawiHEPMNsDboTepfyChdsDv+RTodb1hc0l /orfjg/AGdPxaKgn2LutLrOZRcttSYwGxg1i3aw4YcGNt8azxdXMSGqO FcfKKd4aP7sgz57NClXUSJIjCj5I30iQCQfV9V5idQlKN5MVYUBL7/Pd XghhlQ==
;; Received 1035 bytes from 198.41.0.4#53(a.root-servers.net) in 5764 ms

edu.tw.			3600	IN	NS	moestar.edu.tw.
edu.tw.			3600	IN	NS	moemoon.edu.tw.
edu.tw.			3600	IN	NS	b.twnic.net.tw.
edu.tw.			3600	IN	NS	a.twnic.net.tw.
edu.tw.			3600	IN	NS	c.twnic.net.tw.
edu.tw.			3600	IN	NS	d.twnic.net.tw.
edu.tw.			300	IN	DS	40234 8 1 5A8AB67C461F4330D146EE4E2E5A08CE279B7BEB
edu.tw.			300	IN	DS	40234 8 2 289D061D208C871915EB07F63FB175B21022422D5365D4E945BCE397 104A9C08
edu.tw.			300	IN	RRSIG	DS 8 2 300 20180909000033 20180810000033 46080 tw. QFQiVnzdUorHMrT5IpZTYWHdH0QtEknww+Q+Ql2K9+MFwHhsNqxvk1Uw iSMdQQ9rQ92Tdql99Myv9QGrOxHWd+Ol/ULOg4I2vKIHPqskAN2F7pu1 VhWBAWQ2Tti/pArvGsAn7IaI8DuWWEoD44NjIIOo0NY9YVssz1UUvWpk PU4=
;; Received 691 bytes from 203.73.24.25#53(a.dns.tw) in 259 ms

ksu.edu.tw.		300	IN	NS	dns1.ksu.edu.tw.
ksu.edu.tw.		300	IN	NS	dns3.twaren.net.
ksu.edu.tw.		300	IN	NS	dns2.ksu.edu.tw.
2JL30RTHBS2AC6H8PKQTIF3OCINIARDM.edu.tw. 300 IN	NSEC3 1 0 10 9FCD30FFAD75 2KI9R34PM8A1VO5V97VNHJNVJSL0R6ON NS
2JL30RTHBS2AC6H8PKQTIF3OCINIARDM.edu.tw. 300 IN	RRSIG NSEC3 8 3 300 20180813044341 20180809035735 56424 edu.tw. ebq+txlgqrK4GsZgFoZMTBUgKGcdaiXJtj2JFvKXlLLjdoJyIFC9bEsf aZSoLLfpFIBsF6qons5Yu0YaTp7ypmbMQWws9UOBs9kNnBW++Eaq97Hb RnCq8smJIYnRrmIClEs1kUAkqNEtf9s/TqmcGu3+TLswx5+tVSwbR3DX MIM=
;; Received 388 bytes from 192.83.166.9#53(a.twnic.net.tw) in 1274 ms

www.ksu.edu.tw.		3600	IN	A	120.114.100.65
ksu.edu.tw.		3600	IN	NS	dns3.twaren.net.
ksu.edu.tw.		3600	IN	NS	dns1.ksu.edu.tw.
ksu.edu.tw.		3600	IN	NS	dns2.ksu.edu.tw.
;; Received 202 bytes from 211.79.61.47#53(dns3.twaren.net) in 333 ms

从以上步骤可以看到.114.114.114.114这台DNS服务器依次向以下DNS查询:
root.(全世界一共有13台DNS根服务器)--->tw.---->edu.tw.--->ksu.edu.tw.
最后查询到了主机www.ksu.edu.tw.的IP地址
```

---

#### DNS正向解析字段解析

dig对主机进行正向解析的结果有固定的格式和字段.下面解析一下各字段代表的意义

```
[root@localhost ~]$dig www.baidu.com
#前面省略
;; QUESTION SECTION:
;www.baidu.com.			IN	A

;; ANSWER SECTION:
www.baidu.com.		919	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	209	IN	A	61.135.169.125
www.a.shifen.com.	209	IN	A	61.135.169.121

;; Query time: 17 msec
;; SERVER: 114.114.114.114#53(114.114.114.114)
;; WHEN: Fri Aug 10 14:41:26 CST 2018
;; MSG SIZE  rcvd: 101
```

dig不加任何参数的情况下是查询A(address)记录,也就是主机的IP地址.上面输出结果格式简化如下:

```
[domain]   [ttl]          IN [[RR type]  [RR data]]
[待查数据] [暂存时间(秒)]   IN [[资源类型] [资源内容]]
```

**domain**: 查询的主机名,最好是用FQDN完全合格域名,也就是域名后面要加一个小数点. 代表根.例如:www.baidu.com. (不要忽略最后的一个.) 可以看到ANSWER SECTION字段的www.baidu.com.结尾有个小数点

**ttl:** time to live.意思就是当这笔记录被其他 DNS 服务器查询到后， 这个记录会保持在对方 DNS 服务器的快取中，保持多少秒钟的意思.所以，当你反复执行 dig 之后，就会发现这个时间会减少！为什么呢？因为在你的 DNS 快取中，这笔数据能够保存的时间会开始倒数， 当这个数字归零后，下次有人再重新搜寻这笔记录时，你的 DNS 就会重新沿着 . (root) 开始重来搜寻一遍， 而不会从快取里面捉取了 (因为快取内的资料会被舍弃)。 

**IN**: 这个关键字是固定的.

**RR type:** 这个表示查询类型.这里是查询A记录.

**RR data:** 查询结果.在这里是查询出来的IP地址

```

# 常见的正解文件 RR 相关信息
[domain]    IN  [[RR type]  [RR data]]
主机名.   IN  A           IPv4 的 IP 地址
主机名.   IN  AAAA        IPv6 的 IP 地址
领域名.   IN  NS          管理这个领域名的服务器主机名字.
领域名.   IN  SOA         管理这个领域名的七个重要参数(容后说明)
领域名.   IN  MX          顺序数字  接收邮件的服务器主机名字
主机别名.   IN  CNAME       实际代表这个主机别名的主机名字.
```

---

上面演示了查询IP记录的方法.下面是查询其他类型的方法:

+ **NS:查询管理领域名 (zone) 的服务器主机名** 

如果你想要知道 www.haoshiqi.net 的主机由哪部 DNS 服务器提供的，那就得要使用 NS (NameServer) 的 RR 类型标志来查询。不过，由于 NS 是管理整个领域的，因此，你得要查询的目标将得输入 domain，亦即 haoshiqi.net才行 

```
[root@localhost ~]$dig -t ns haoshiqi.net

;; QUESTION SECTION:
;haoshiqi.net.			IN	NS

;; ANSWER SECTION:
haoshiqi.net.		1720	IN	NS	ns4.dnsv3.com.
haoshiqi.net.		1720	IN	NS	ns3.dnsv3.com.
#可以看到这个域名的DNS服务商是ns4.dnsv3.com.和ns3.dnsv3.com.
```

---

+ **SOA ：查询管理领域名的服务器管理信息** 

如果你有多部 DNS 服务器管理同一个领域名时，那么最好使用 master/slave 的方式来进行管理。既然要这样管理， 那就得要宣告被管理的 zone file 是如何进行传输的，此时就得要 SOA (Start Of Authority) 的标志了。 

```
[root@localhost ~]$dig -t soa haoshiqi.net

;; QUESTION SECTION:
;haoshiqi.net.			IN	SOA

;; ANSWER SECTION:
haoshiqi.net.		600	IN	SOA	ns3.dnsv3.com. enterprise1dnsadmin.dnspod.com. 1530265971 3600 180 1209600 180
```

SOA 主要是与领域有关，所以前面当然要写 ksu.edu.tw 这个领域名。而 SOA 后面共会接七个参数，这七个参数的意义依序是：

+ Master DNS 服务器主机名：这个领域主要是哪部 DNS 作为 master 的意思。在本例中， ns3.dnsv3.com 为 主要 DNS 服务器；

+ 管理员的 email：那么管理员的 email 为何？发生问题可以联络这个管理员。

+ 序号 (Serial)：这个序号代表的是这个数据库档案的新旧，序号越大代表越新。 当 slave 要判断是否主动下载新的数据库时，就以序号是否比 slave 上的还要新来判断，若是则下载，若不是则不下载。 所以当你修订了数据库内容时，记得要将这个数值放大才行！ 为了方便用户记忆，通常序号都会使用日期格式『YYYYMMDDNU』来记忆

+ 更新频率 (Refresh)：那么啥时 slave 会去向 master 要求数据更新的判断？ 就是这个数值定义的。昆山科大的 DNS 设定每 3600 秒进行一次 slave 向 master 要求数据更新。那每次 slave 去更新时， 如果发现序号没有比较大，那就不会下载数据库档案。

+ 失败重新尝试时间 (Retry)：如果因为某些因素，导致 slave 无法对 master 达成联机， 那么在多久的时间内，slave 会尝试重新联机到 master。在本例中，180秒会重新尝试一次。意思是说，每 180秒 slave 会主动向 master 联机，但如果该次联机没有成功，那接下来尝试联机的时间会变成 180秒。若后来有成功，则又会恢复到 180 秒才再一次联机。

+ 失效时间 (Expire)：如果一直失败尝试时间，持续联机到达这个设定值时限， 那么 slave 将不再继续尝试联机，并且尝试删除这份下载的 zone file 信息。这设定为 1209600秒。意思是说，当联机一直失败，每 180秒尝试到达 1209600 秒后，slave 将不再更新，只能等待系统管理员的处理。

+ 快取时间 (Minumum TTL)：如果这个数据库 zone file 中，每笔 RR 记录都没有写到 TTL 快取时间的话，那么就以这个 SOA 的设定值为主。

  ---

+ **CNAME ：设定某主机名的别名 (alias)**

有时候你不想要针对某个主机名设定 A 的标志，而是想透过另外一部主机名的 A 来规范这个新主机名时， 可以使用别名 (CNAME) 的设定 

```
[root@localhost ~]$dig www.baidu.com
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.baidu.com.			IN	A

;; ANSWER SECTION:
www.baidu.com.		328	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	34	IN	A	61.135.169.125
www.a.shifen.com.	34	IN	A	61.135.169.121
```

意思是说，当你要追查www.baidu.com 时，请找 www.a.shifen.com.那个主机，而那个主机的 A 就上面第二行的显示了。  

这个 CNAME 有啥好处呢？用 A 就好了吧？其实还是有好处的，举例来说，如果你有一个 IP，这个 IP 是给很多主机名使用的。 那么当你的 IP 更改时，所有的数据就得通通更新 A 标志才行。如果你只有一个主要主机名设定 A，而其他的标志使用 CNAME 时，那么当 IP 更改，那你只要修订一个 A 的标志，其他的 CNAME 就跟着变动了！处理起来比较容易啊！ 

---

+ **MX ：查询某领域名的邮件服务器主机名** 

MX 是 Mail eXchanger (邮件交换) 的意思，通常你的整个领域会设定一个 MX ，代表，所有寄给这个领域的 email 应该要送到后头的 email server 主机名上头才是 

---

#### 反向解析RR数据

在讲反解之前，先来谈谈正解主机名的追踪方式。以 www.api.haoshiqi.net. 来说，整个网域的概念来看， 越右边出现的名称代表网域越大！举例来说，.(root) > net > haoshiqi 以此类推。因此追踪时，是由大范围找到小范围， 

但是 IP 则不一样啊！以我们的114.55.224.232 来说好了，当然是 114 > 55 > 224 > 232 ，左边的网域最大！ 与预设的 DNS 从右边向左边查询不一样啊！那怎办？为了解决这个问题，所以反解的 zone 就必须要将 IP 反过来写，而在结尾时加上 .in-addr.arpa. 的结尾字样即可。所以，当你想要追踪反解时，那么反解的结果就会是： 

```
[root@localhost ~]$dig -x 114.55.224.232

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;232.224.55.114.in-addr.arpa.	IN	PTR

;; AUTHORITY SECTION:
55.114.in-addr.arpa.	225	IN	SOA	rdns1.alidns.com. dnsmgr.alibaba-inc.com. 2015011323 1800 600 1814400 300
```

PTR就是反向解析的意思.要注意的就是 zone 的名称了！要将 IP 反转过来写，并且结尾加上 .in-addr.arpa. 才行 

