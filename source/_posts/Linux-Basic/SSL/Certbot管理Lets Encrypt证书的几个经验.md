---
title: Certbot管理Lets Encrypt证书的几个经验  
date: 2018-10-15 17:59:58  
tags:  SSL
categories: [Linux-Basic,SSL ]  
comments: true  
copyright: true  

---

## Certbot管理Lets Encrypt证书的几个经验

certbot提供了很多命令和插件申请,注销,续约letsencrypt的证书.使用非常方便.借鉴网上的一些小技巧,整理出了这篇文章.

后续如果有更多教训或者经验技巧,还会更新此篇文章.

<!--more-->
--

### 1.设置邮箱接收证书过期的通知

一张Let's Encrypt 证书有效期是 90 天，有效期太短，Let's Encrypt 会在证书快过期的时候，给 email地址发送证书快过期的通知。通过下列命令设置或者修改一个email地址:

```
$ certbot-auto register --update-registration --email admin@example.com
```
--

### 2.安装certbot-auto

certbot 和 certbot-auto 本质上没有区别，安装certbot-auto也非常简单:

```
$ wget https://dl.eff.org/certbot-auto
$ chmod a+x ./certbot-auto
```
而如果是通过apt-get,yum等安装方式的话.安装的就是certbot工具

```
$ apt-get install certbot
```
--

### 3.查看申请了哪些证书

通过下列命令可以查看目前服务器上申请过的证书,以及证书所在路径,证书有效期等信息:

```
root@docker:~# certbot certificates

Found the following certs:
  Certificate Name: topic.dev.doweidu.com
    Domains: topic.dev.doweidu.com
    Expiry Date: 2019-01-09 08:22:20+00:00 (VALID: 85 days)
    Certificate Path: /etc/letsencrypt/live/topic.dev.doweidu.com/fullchain.pem
    Private Key Path: /etc/letsencrypt/live/topic.dev.doweidu.com/privkey.pem
```
--

### 3.注销证书

如果域名证书不需要的话,可以将证书注销掉,但是最好不要使用rm直接删除证书文件.最好是用revoke选项通知LetsEncrypt注销证书.

注销证书命令如下:

```
root@1264f80bcd97:~# ./certbot-auto revoke --cert-path /etc/letsencrypt/live/deploy.haoshiqi.net/cert.pem  --reason superseded
```
--cert-path表示证书文件所在路径
--reason表示注销原因.superseded表示作废的意思

输出如下:

```
root@1264f80bcd97:~# ./certbot-auto revoke --cert-path /etc/letsencrypt/live/deploy.haoshiqi.net/cert.pem  --reason superseded
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you like to delete the cert(s) you just revoked?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es (recommended)/(N)o: y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Deleted all files relating to certificate deploy.haoshiqi.net.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Congratulations! You have successfully revoked the certificate that was located
at /etc/letsencrypt/live/deploy.haoshiqi.net/cert.pem
```


