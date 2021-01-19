---
title: Openvpn客户端无法连接 OpenSSL: error
date: 2020-09-22 22:59:58
tags:  [Linux,openvpn ]
categories: Linux-Service
comments: true
copyright: true
---



## Openvpn客户端无法连接 OpenSSL: error

### 背景

今天阿里云的Openvpn服务器部署了docker服务后,需要升级内存配置.服务器升级重启后,发现客户端无法连接Openvpn.

---

### 故障表现

在openvpn客户端日志中发现下面报错信息:

```
2020-09-22 16:00:49 WARNING: No server certificate verification method has been enabled.  See http://openvpn.net/howto.html#mitm for more info.
```

openvpn服务端日志报错信息:

```
ue Sep 22 15:47:49 2020 27.115.51.166:26184 TLS: Initial packet from [AF_INET]27.115.51.166:26184, sid=c0cb2b12 4b3187b2
Tue Sep 22 15:47:49 2020 27.115.51.166:26184 VERIFY ERROR: depth=0, error=CRL has expired: CN=xxxxxx
Tue Sep 22 15:47:49 2020 27.115.51.166:26184 OpenSSL: error:14089086:SSL routines:ssl3_get_client_certificate:certificate verify failed
Tue Sep 22 15:47:49 2020 27.115.51.166:26184 TLS_ERROR: BIO read tls_read_plaintext error
Tue Sep 22 15:47:49 2020 27.115.51.166:26184 TLS Error: TLS object -> incoming plaintext read error
Tue Sep 22 15:47:49 2020 27.115.51.166:26184 TLS Error: TLS handshake failed
Tue Sep 22 15:47:49 2020 27.115.51.166:26184 SIGUSR1[soft,tls-error] received, client-instance restarting
Tue Sep 22 15:48:06 2020 27.115.51.166:33480 TLS: Initial packet from [AF_INET]27.115.51.166:33480, sid=11b9760e 97f6d068
Tue Sep 22 15:48:07 2020 27.115.51.166:33480 VERIFY ERROR: depth=0, error=CRL has expired: CN=xxxxxx
```

**日志关键字**

<!--more-->

```
OpenSSL: error:14089086:SSL routines:ssl3_get_client_certificate:certificate verify failed
```

---

### 排查

服务器重启后需要注意的几个问题:

1.检查以下几个服务是否启动:

```
systemctl status openvpn@server
systemctl status iptables
```

2.由于docker服务会初始化iptables,所以docker启动后需要手动添加一条iptables规则

```
iptables -t nat -A POSTROUTING -s 10.111.255.0/24 -o eth0 -j MASQUERADE
```

3.检查ip转发功能

```
vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
```

4.检查阿里云的安全组规则,是否开通了1094的udp协议

---

### 解决

最终解决方案还是要靠google解决,在openvpn的wiki上找到了解决方案,

具体网站链接:[openvpn wiki](https://community.openvpn.net/openvpn/wiki/CertificateRevocationListExpired?__cf_chl_jschl_tk__=40e85ba16653b7db84d828db819eafbc2b5a9faf-1600761449-0-AUAdZiIXwLUqBJDJAcDe9htVtUTZlGJm8m_RYLUsxLu2he8Myk5WXwzQn-CZdZyBDJRHn9clM-6y0ITsnKk0Pru3AwB7EOc0LhjyrV9unNnBv0V9_skxNC2n3per9e1TQJjcmmtwnaNl23Sp5D8p9FZYyX5PO-vtkdp1i7dyh_x1KSFwZqibI8Zt4saVoABGkfMJ3nKeUJpIIlnEhRGhfJrQwQZqvvG6EAS1CJUHRR8uqKNyqmEz90RmqcLGc8ytoOTIBYhJMs8OsPk_dA2nKA47QObzI5-4SEUZkC5ZaxhNCfkRBGx9kwimWfBJFtxJkPzI5_vkXONWXhoB0wi5cfE)

解决问题很简单:

```
#需要进入到下面这个目录下
cd /etc/openvpn/easy-rsa/3

#更新一下crl文件.
[root@dwd-Dnsmasq 3]# ./easyrsa gen-crl

Using configuration from ./openssl-1.0.cnf

An updated CRL has been created.
CRL file: /etc/openvpn/easy-rsa/3/pki/crl.pem
```



