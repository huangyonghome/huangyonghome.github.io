---
title: 使用acme.sh申请https证书
date: 2018-10-15 11:59:58
tags:  SSL
categories: [Linux-Basic,SSL ]
comments: true
copyright: true
---



### 使用acme.sh申请https证书

### 介绍

**acme.sh** 实现了 `acme` 协议, 可以从 letsencrypt 生成免费的证书.

acme是一个shell脚本。安装使用都很方便。不依赖于python或官方的Let's Encrypt客户端。
只需一个脚本即可自动颁发，续订和安装证书。 不需要root/sudoer访问权限。

acme的github项目有详细的中文文档：[acme.sh](https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E)

---

### 对比certbot工具

还有另外一个工具**certbot-auto**也可以申请letsencrypt免费证书。

certbot工具需要安装一大堆系统库以及 Python 库，Python 的 pip 在国内还会有墙的问题…而且每次certbot工具自动更新版本的时候都会面临pip被墙的风险。所以很难使用。

当然，**acme.sh**相比certbot更强大的地方还在于：

1.acme支持dns认证方式

2.acme带有自动更新dns记录，验证dns的脚本(需要dns服务商的api token)

3.支持通配符证书的申请

4.acme安装后就自动写入crontab记录，意味着只要证书申请后就自动更新，完全不需操心

<!--more-->

---

### acme配置证书步骤

acme配置证书相对比cerbot工具复杂很多。需要经过以下几个步骤

1.申请证书（此时证书默认在家目录的.acme目录下）

2.创建目标文件夹存放证书

3.安装证书到目标文件夹

4.配置nginx

---

### 配置步骤

这个文档主要对dns验证方式和通配符域名证书申请进行说明。单域名的网站文件认证方式其实大同小异。

所以此文档不赘述acme.sh工具的安装,基础使用等.详细信息可以去github上看文档

另外由于我们使用的dns服务商是dnspod.所以这里以dnspod为例.

一.**在dnspod上创建api token**

创建完成后,记录ID和TOKEN.然后在服务器上导入下列2个环境变量:

```
export DP_Id="数字ID"
export DP_Key="字符串key"
```

**二.给acme.sh工具添加一个alias**

```
alias acme.sh=~/.acme.sh/acme.sh
```

**三.申请一个通配符证书**.比如.beta.haoshiqi.net域名.

dns_dp表示使用dnsapi/dns_dp.sh脚本来更新和验证DNSPOD服务商的dns解析

```
acme.sh --issue --dns dns_dp -d beta.haoshiqi.net -d *.beta.haoshiqi.net
```

acme.sh在dnsapi目录下提供了大量不同的shell脚本来针对不同的dns服务商.在github上可以看到具体每家dns解析服务商所对应的—dns 脚本文件: [dnsapi](https://github.com/Neilpang/acme.sh/wiki/dnsapi)

**四.新建一个以域名命名的目录用于安装证书**

```
mkdir -pv /data/letsencrypt/beta.haoshiqi.net
```

**五.安装证书到指定目录**

```
acme.sh  --installcert  -d  beta.haoshiqi.net  \
	     --key-file /data/letsencrypt/beta.haoshiqi.net/beta.haoshiqi.net.key  \
	     --fullchain-file /data/letsencrypt/beta.haoshiqi.net/fullchain.cer     \
	     --reloadcmd  "supervisorctl -c /etc/supervisord/supervisord.conf restart nginx"
```

> reloadcmd 是用来重载证书文件的命令.nginx -s reload并不会重载证书配置

**六.配置nginx**

```
    listen 443 ssl http2; 
    ssl_certificate  /data/letsencrypt/beta.haoshiqi.net/fullchain.cer;
    ssl_certificate_key /data/letsencrypt/beta.haoshiqi.net/beta.haoshiqi.net.key;
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
```

> include 和 ssl_dhparam配置指令所对应的文件是我从certbot工具生成的文件中拷贝过来的.
>
> 其中options-ssl-nginx.conf包含兼容的tls协议版本,认证方法等信息
>
> ssl-dhparams.pem包含密钥算法信息

