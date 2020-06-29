---
title: lets encrypt学习
date: 2018-06-24 11:59:58
tags:  SSL
categories: [Linux-Basic,SSL ]
comments: true
copyright: true

---



## lets encrypt学习

[官网介绍](https://letsencrypt.org/)

[certbot官网](https://certbot.eff.org/)

#### 一.letsencrypt介绍: 

LetsEncrypt是一个CA( Certificate Authority ).颁发证书给某个域名.为了能从letsencrypt拿到证书,必须要能证明此域名在你的web服务器上.推荐在shell环境下使用Certbot ACME客户端获取letsencrypt证书 

<!--more-->

#### 二.letsencrypt工作原理: 

Letsencrypt和ACME协议的目标是用户可以架设HTTPS服务器,并且能自动获得浏览器信赖的证书.在web server上运行的证书管理agent可以实现这些需求 

为了理解letsencrypt工作过程,用Https://example.com/来举例.主要经过以下2个过程

1.agent向CA证明这台web服务器拥有一个域名.

2.agnet为此域名查询,申请,撤销证书 

**域名确认:**

Let's Encrypt使用public key来验证服务器管理员.在agent和Lets encrypt第一次交互时,会生成一个新的Key对来向Lets encrypt证明这台web服务器配置了一个或者多个域名.这有点类似于传统的CA要求创建一个账户,并且将域名添加到账户上. 

agent会询问CA如何才能证明自己拥有example.com域名.Let's Encrypt会查看example.com域名,发布一个或者多个设置配置.例如,CA会要求agent:在example.com域名下提供一个DNS记录.或者:在example.com URI下提供一个http资源 

另外,lets encypt CA会提供一个临时场所,要求agent签署它的私钥 

![letsencrypt](https://img1.jesse.top/static/images/Linux/letsencrypt.png)


Agent会在example.com站点上的一个指定路径内创建一个文件.同时签署自己私钥.一旦agent完成了这些配合工作.它会通知CA表示它已经完成了域名确认工作. 

CA接下来会去检查agent是否完成了相关设置,CA会验证agent的私钥,并且尝试去该站点下载agent创建的文件.如果一切正常,由该公钥鉴定的agent会被授权为example域名的证书管理者, 

![letsencrypt](https://img1.jesse.top/static/images/Linux/letsencrypt1.png)

---

**证书的发行和撤销:** 

一旦agent拥有被授权的KEY对,查询,申请和撤销证书就变的很简单----直需要发送证书管理消息,并且使用经过授权的KEY对签署即可为该域名申请一个证书的步骤, 

1.agent构建一个Certificate Signing Request(CSR),要求lets encrypt CA发为该域名发布一个证书.通常情况下,CSR包括一个私钥的签名和一个对应的公钥,Agent也会使用授权过的秘钥签署整个CSR.这样一来,CA便会知道它已经被授权过了.

2.当CA收到了请求,会检查签名.然后为域名发送一个证书 

![letsencrypt](https://img1.jesse.top/static/images/Linux/letsencrypt2.png)


撤销的工作有点类似.agent签署一个撤销请求,当CA接受到请求,并且经过验证后,CA会发布撤销信息到正常撤销渠道(例如:CRLs,OCSP).所以网页浏览器便不会接受被撤销的证书 

![letsencrypt](https://img1.jesse.top/static/images/Linux/letsencrypt3.png)
