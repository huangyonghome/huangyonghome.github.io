---
title: letsencrypt证书上传到阿里云SLB负载均衡
date: 2018-10-15 11:59:58
tags:  SSL
categories: [Linux-Basic,SSL ]
comments: true
copyright: true

---

## letsencrypt证书上传到阿里云SLB负载均衡

### 简介
最近在阿里云开通了一个SLB负载均衡实例,然后把ECS上申请的免费的letsencrypt证书上传到SLB下.

按照简书上的这个流程: [letsencrypt证书上传到阿里云SLB](https://www.jianshu.com/p/b1814f200fef)

但是出现了一个奇怪的现象.浏览器,PC,IOS等设备Https访问均正常,但是所有的安卓设备微信扫码小程序登陆却提示证书错误:**err:request:fail ssl hand shake error**
<!--more-->
---

### SSL诊断

使用SSL在线诊断工具[SSL检测](http://aq.chinaz.com/) .可以看到确实是证书存在一些问题.在网上baidu和google搜寻也没有太多可用的资料.

但是我将该域名解析到ECS本地(SLB上的letsencrypt证书就是该ECS上申请的).再次利用SSL在线诊断工具,发现证书一切正常.安卓访问该域名也没有问题.

于是,初步判断是证书上传到SLB后出现的问题.

---

### 解决方案

根据阿里云的文档[SSL要求](https://help.aliyun.com/document_detail/32332.html).letsencrypt是中级证书颁发机构,申请下来的证书包含了多个证书文件.其中包含以下主要的4个文件:

```
[root@DWD-BETA m.betaapi.haoshiqi.net]# ls
cert.pem  chain.pem  fullchain.pem  privkey.pem  README
```

cert.pem                #服务器证书

chain.pem     	 #浏览器需要的所有证书但不包括服务端证书，比如根证书和中间证书

fullchain.pem 	#中级证书,包括了cert.pem和chain.pem的内容

privkey.pem    	#服务器证书私钥



由于刚才只将cert.pem服务器证书和privkey.pem服务器证书私钥文件上传到SLB.所以出现证书错误的异常情况.

---

### 解决

根据上面阿里云的文档,在阿里云SLB上创建证书时,将cert.pem证书和fullchain.pem证书内容复制到"公钥证书"栏中.

注意:服务器证书(cert.pem)放第一位，中级证书(fullchain.pem)放第二位，中间不能有空行。

接下来将privkey.pem私钥证书放在"私钥"一栏中.

