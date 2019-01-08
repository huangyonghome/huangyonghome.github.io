---
title: letsencrypt配置通配符域名证书
date: 2018-10-15 11:59:58
tags:  SSL
categories: [Linux-Basic,SSL ]
comments: true
copyright: true

---

## letsencrypt配置通配符域名证书

### 介绍
在今年稍早些的时候,letsencrypt终于开始免费支持通配符域名证书了.但是只支持一个层级的通配主机域名.

比如我曾经踩过的坑:*.dev.xxx.com通配符域名证书并不能支持api.v3.dev.xxx.com这种三层子域名.

如果需要为api.v3.dev.xxx.com申请通配符证书,则还需要申请\*.v3.dev.xxx.com通配符证书

<!--more-->
--

### letsencryp通配符证书申请前提

#### ACME V2版本要求

为了实现通配符证书，Let’s Encrypt 对 ACME 协议的实现进行了升级，只有 v2 协议才能支持通配符证书。

也就是说任何客户端只要支持 ACME v2 版本，就可以申请通配符证书了，是不是很激动。

可以查看下自己惯用的客户端是不是支持 ACME v2 版本，官方介绍 Certbot 0.22.0 版本支持新的协议版本

```
root@docker:~# ./certbot-auto --version
certbot 0.27.1
```
#### 验证域名所有权的方式

Let’s Encrypt颁发证书的时候，需要校验域名的所有权，证明操作者有权利为该域名申请证书，目前支持三种验证方式：

dns-01：给域名添加一个 DNS TXT 记录。  
http-01：在域名对应的 Web 服务器下放置一个 HTTP well-known URL 资源文件。  
tls-sni-01：在域名对应的 Web 服务器下放置一个 HTTPS well-known URL 资源文件。  

**而申请通配符证书，只能使用 dns-01 的方式**

--

### 申请通配符证书方法

```
./certbot-auto certonly  -d *.devapi.haoshiqi.net --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory 
```

介绍下相关参数：

* certonly，表示安装模式，Certbot 有安装模式和验证模式两种类型的插件。  
* --manual 表示手动安装插件，Certbot 有很多插件，不同的插件都可以申请证书，用户可以根据需要自行选择  
* -d 为那些主机申请证书，如果是通配符，输入 *.devapi.haoshiqi.net（可以替换为你自己的域名）  
* --preferred-challenges dns，使用 DNS 方式校验域名所有权  
* --server，Let's Encrypt ACME v2 版本使用的服务器不同于 v1 版本，需要显示指定。  

下面是命令输出:

```
root@docker:~# ./certbot-auto certonly  -d *.devapi.haoshiqi.net --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for devapi.haoshiqi.net

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.devapi.haoshiqi.net with the following value:

pjTQy43PDXAXBdP3roiftl1uzO-BBidaIG45703ReGs

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

首先,会要求是否同意IP和证书绑定. 输入y继续下一步

其次,会验证DNS的TXT记录.这里非常关键,先不要回车.Letsencrypt要求配置 DNS TXT 记录，从而校验域名所有权，也就是判断证书申请者是否有域名的所有权。  

上面要求给_acme-challenge.devapi.haoshiqi.net配置一条TXT记录.记录值为:pjTQy43PDXAXBdP3roiftl1uzO-BBidaIG45703ReGs

我们使用的是DnsPort域名解析服务也就是腾讯DNS运营商.在域名的DNS解析记录里添加一条主机记录:_acme-challenge.devapi,对应上面的txt值即可.具体配置过程就不再介绍了

配置完成后,先检查一下配置是否生效:

```
 huangyong@huangyong-Macbook-Pro  ~  dig -t txt _acme-challenge.devapi.haoshiqi.net @8.8.8.8

; <<>> DiG 9.10.6 <<>> -t txt _acme-challenge.devapi.haoshiqi.net @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3656
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;_acme-challenge.devapi.haoshiqi.net. IN	TXT

;; ANSWER SECTION:
_acme-challenge.devapi.haoshiqi.net. 599 IN TXT	"pjTQy43PDXAXBdP3roiftl1uzO-BBidaIG45703ReGs"
```

可以看到已经生效了.此时再回到刚才的界面,按回车键..输出如下信息:

```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.devapi.haoshiqi.net with the following value:

pjTQy43PDXAXBdP3roiftl1uzO-BBidaIG45703ReGs

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/devapi.haoshiqi.net/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/devapi.haoshiqi.net/privkey.pem
   Your cert will expire on 2019-01-13. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot-auto
   again. To non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

root@docker:~#
```

证书申请成功,证书和秘钥保存在下面的目录

```
root@docker:~# ls -l /etc/letsencrypt/live/devapi.haoshiqi.net/
total 4
lrwxrwxrwx 1 root root  43 Oct 15 15:46 cert.pem -> ../../archive/devapi.haoshiqi.net/cert1.pem
lrwxrwxrwx 1 root root  44 Oct 15 15:46 chain.pem -> ../../archive/devapi.haoshiqi.net/chain1.pem
lrwxrwxrwx 1 root root  48 Oct 15 15:46 fullchain.pem -> ../../archive/devapi.haoshiqi.net/fullchain1.pem
lrwxrwxrwx 1 root root  46 Oct 15 15:46 privkey.pem -> ../../archive/devapi.haoshiqi.net/privkey1.pem
-rw-r--r-- 1 root root 682 Oct 15 15:46 README
```
--

### Nginx配置证书

证书申请下来后,就可以在nginx配置证书了.配置方法比较简单.只需要在nginx配置文件中添加几行命令即可.例如:

```
root@docker:~# vim /data/conf/nginx/conf.d/hsq-openapi.conf

server {
  listen 443 ssl http2;

  server_name m.devapi.haoshiqi.net; # 域名
 
 #添加或者修改以下3行SSL配置:
 
    ssl_certificate /etc/letsencrypt/live/devapi.haoshiqi.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/devapi.haoshiqi.net/privkey.pem;
    ssl_trusted_certificate  /etc/letsencrypt/live/devapi.haoshiqi.net/chain.pem;
```
重启nginx,浏览器访问域名.通配符证书已经生效.如下图所示:

![](http://pabkmteb4.bkt.clouddn.com/letsencrypt.png)

--

### 通配符证书自动续期

letsencrypt的通配符证书的有效期同样为90天.但是通配符的自动续约比较复杂.因为需要人工配置DNS记录来验证域名控制权,所以不能像单域名证书一样使用certbot工具来自动续约了.

不过好在网络上有很多大神写了一些比较优秀的脚本工具来通过调用DNS解析运营商的API接口实现自动配置DNS记录.收集了2个作者的脚本来实现自动续期:

* [本文原作者编写的工具](https://github.com/ywdblog/certbot-letencrypt-wildcardcertificates-alydns-au)
* [acme脚本](https://github.com/Neilpang/acme.sh/tree/master/dnsapi)

--

本文大部分知识点来自于:https://www.jianshu.com/p/c5c9d071e395

非常感谢原文作者!






