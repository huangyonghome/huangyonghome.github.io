---
title: Centos7 搭建Anyconnect VPN
date: 2018-07-13 14:59:58
tags:  [Linux,anyconnect ]
categories: Linux-Service
comments: true
copyright: true
---



## Centos7 搭建Anyconnect VPN

### 简介

之前在公司内部已经搭建了openvpn服务.但是由于openvpn的移动客户端被墙了.所以需要搭建另外一个VPN.

ss(shadowsocks)或者ssr的IOS移动客户端被下架,或者需要收费.无奈放弃

OpenConnet Server（ocserv)通过实现Cisco的AnyConnect协议，用DTLS作为主要的加密传输协议。

官方主页：http://www.infradead.org/ocserv/  

<!--more-->

Anyconnect有以下特点:  

- AnyConnect的VPN协议默认使用UDP作为数据传输，但如果有什么网络问题导致UDP传输出现问题，它会利用最初建立的TCP TLS通道作为备份通道，降低VPN断开的概率。
- AnyConnect作为Cisco新一代的VPN解决方案，被用于许多大型企业

---

### 环境:

KVM虚拟机  
操作系统:centos7.4  
openvpn版本:2.4 (之前搭建,anyconnect VPN和Openvpn在同一台服务器)  
easyrsa版本:3.0  
公司服务器内网地址网段:10.0.0.0/24  
openvpn VPN客户端内网地址段:10.0.80.0/24  
anyconnect VPN客户端内网地址段:10.0.81.0/24  

### 安装步骤

参考教程:  
[在 CentOS 7 上搭建 Cisco AnyConnect VPN](https://ifreedom.one/2015/04/20/Setup-Cisco-AnyConnect-VPN-on-CentOS7/)  
[架设OpenConnect Server](https://bitinn.net/11084/)

一.yum安装

```
yum install epel-release
yum install ocserv
```

二.创建一个工作目录用来生成证书

```
mkdir /etc/anyconnect
cd /etc/anyconnect
```

三.生成CA证书.并且拷贝到/etc/ocserv目录下

```
[root@openvpn anyconnect]$ certtool --generate-privkey --outfile ca-key.pem
[root@openvpn anyconnect]$ cat > ca.tmpl <<EOF
cn = "VPN CA"
organization = "DWD"
serial = 1
expiration_days = 3650
ca
signing_key
cert_signing_key
crl_signing_key
EOF

[root@openvpn anyconnect]$ certtool --generate-self-signed --load-privkey ca-key.pem \
--template ca.tmpl --outfile ca-cert.pem

[root@openvpn anyconnect]$ cp ca-cert.pem /etc/ocserv/

```

四.生成本地服务器证书,并且拷贝到/etc/ocserv目录下

```
[root@openvpn anyconnect]$ certtool --generate-privkey --outfile server-key.pem

[root@openvpn anyconnect]$ cat >server.tmpl <<EOF
cn = "dwd.com"
organization = "DWD"
serial = 2
expiration_days = 3650
encryption_key
signing_key
tls_www_server
EOF

[root@openvpn anyconnect]$ certtool --generate-certificate --load-privkey server-key.pem \
--load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem \
--template server.tmpl --outfile server-cert.pem

[root@openvpn anyconnect]$ cp server-cert.pem server-key.pem /etc/ocserv/
```
---

### 配置ocserv

Ocserv提供了多种认证登录方式.主要有:
- pam本地系统账户
- ocsrev创建的明文账户(需要指定passwd密码文件.下面我指定的是/etc/ocserv/ocpasswd)
- certificate证书认证
- redius认证

```
[root@openvpn anyconnect]$grep -A 5 "#auth" /etc/ocserv/ocserv.conf

#auth = "pam"
#auth = "pam[gid-min=1000]"
#auth = "plain[passwd=/etc/ocserv/ocpasswd]"
auth = "certificate"
#auth = "radius[config=/etc/radiusclient/radiusclient.conf,groupconfig=true]"
```

本教程主要讨论明文密码认证和证书认证方式

#### 证书认证方式

> 据我测试,证书认证方式在window,IOS,Android设备都正常.但是在MAC电脑,MAC笔记本使用时,虽然导入且信任了证书,但是登录VPN时并没有使用导入的证书.ocserv的服务端提示如下错误:

>
```
Jul 12 02:46:36 openvpn ocserv[14967]: worker:  tlslib.c:488: no certificate was found
Jul 12 02:46:36 openvpn ocserv[14967]: worker: 10.0.99.1 no certificate provided for authentication
Jul 12 02:46:36 openvpn ocserv[14815]: main:10.0.99.1:37650 user disconnected (reason: unspecified, rx: 0, tx: 0)
```
> 此问题至今还未解决,google查找资料时疑似是为修复的Bug.


一.配置ocserv配置文件.如下所示

```
[root@openvpn anyconnect]$sed -e '/^#/d' /etc/ocserv/ocserv.conf | sed '/^$/d'
auth = "certificate"
tcp-port = 4333
udp-port = 4333
run-as-user = ocserv
run-as-group = ocserv
socket-file = ocserv.sock
chroot-dir = /var/lib/ocserv
isolate-workers = true
max-clients = 50
max-same-clients = 10
keepalive = 32400
dpd = 90
mobile-dpd = 1800
switch-to-tcp-timeout = 25
try-mtu-discovery = true
server-cert = /etc/ocserv/server-cert.pem
server-key = /etc/ocserv/server-key.pem
ca-cert = /etc/ocserv/ca-cert.pem
cert-user-oid = 2.5.4.3
tls-priorities = "NORMAL:%SERVER_PRECEDENCE:%COMPAT:-VERS-SSL3.0"
auth-timeout = 240
min-reauth-time = 300
max-ban-score = 50
ban-reset-time = 300
cookie-timeout = 300
deny-roaming = false
rekey-time = 172800
rekey-method = ssl
use-occtl = true
pid-file = /var/run/ocserv.pid
device = vpns
predictable-ips = true
default-domain = example.com
ipv4-network = 10.0.81.0
ipv4-netmask = 255.255.255.0
dns = 114.114.114.114
ping-leases = false
route = 10.0.0.0/255.255.255.0
cisco-client-compat = true
dtls-legacy = true
user-profile = profile.xml
```

关键配置参数解析:

```
#认证方式,这里使用了certificate证书认证  
auth = "certificate"  
#最大客户端连接数    
max-clients = 50   
#同一客户端最大同时连接数    
max-same-clients = 10    
#优化VPN速度和稳定性 
try-mtu-discovery = true  
#服务端证书路径  
server-cert = /etc/ocserv/server-cert.pem  
#服务端key路径 
server-key = /etc/ocserv/server-key.pem  
#ca证书路径,如果是证书验证则需要开启这个参数,如果是密码认证,则注释掉  
ca-cert = /etc/ocserv/ca-cert.pem  
# 确保服务器正确读取用户证书（后面会用到用户证书） 
cert-user-oid = 2.5.4.3  
#分发给VPN客户端的IP地址范围,DNS地址  
ipv4-network = 10.0.81.0  
ipv4-netmask = 255.255.255.0  
dns = 114.114.114.114  
#如果仅仅是访问以下内网地址则指定route参数,如果注释所有route参数则表示所有流量走VPN  
route = 10.0.0.0/255.255.255.0  
```

二. 创建客户端证书

编辑创建客户端证书脚本.

```
[root@openvpn anyconnect]$vim gen-client.sh

#!/bin/bash

USER=$1
CA_DIR=$2
SERIAL=`date +%s`

#生成客户端key
certtool --generate-privkey --outfile $USER-key.pem

#生成证书模板文件
cat << _EOF_ >user.tmpl
cn = "$USER"
unit = "users"
serial = "$SERIAL"
expiration_days = 9999
signing_key
tls_www_client
_EOF_

#生成用户证书
certtool --generate-certificate --load-privkey $USER-key.pem --load-ca-certificate $CA_DIR/ca-cert.pem --load-ca-privkey $CA_DIR/ca-key.pem --template user.tmpl --outfile $USER-cert.pem

#将证书转换成p12格式,以便客户端导入证书
openssl pkcs12 -export -inkey $USER-key.pem -in $USER-cert.pem -name "$USER VPN Client Cert" -certfile $CA_DIR/ca-cert.pem -out $USER.p12
```
> 官网使用的是certtool命令将证书转换成p12格式:

```
certtool --to-p12 --load-privkey user-key.pem --pkcs-cipher 3des-pkcs12 --load-certificate user-cert.pem --outfile user.p12 --outder
```

给脚本赋权
```
chomod +x gen-client.sh
```

创建用户文件.例如给我创建一个客户端证书

```
[root@openvpn anyconnect]$ mkdir huangyong
[root@openvpn anyconnect]$cd huangyong

#脚本的$1参数表示创建的用户名,$2参数表示ca证书位置.
#按提示给证书设置一个密码(建议).也可以空密码(MAC电脑不支持导入空密码证书).
[root@openvpn huangyong]$../gen-client-cert.sh huangyong ..


#脚本执行完成后,在用户文件夹可以看到证书文件:
[root@openvpn huangyong]$ll
total 20
-rw-r--r--. 1 root root 1176 Jul 10 22:47 huangyong-cert.pem
-rw-------. 1 root root 5826 Jul 10 22:47 huangyong-key.pem
-rw-r--r--. 1 root root 3376 Jul 10 22:47 huangyong.p12
-rw-r--r--. 1 root root  104 Jul 10 22:47 user.tmpl
```

三. 开启一个apache或者nginx服务.以便客户端可以访问并且导入服务端证书.这里我使用Yum安装了一个nginx服务.

3.1 编写一个nginx配置文件:  

```
[root@openvpn huangyong]$cd /etc/nginx/conf.d/
[root@openvpn conf.d]$vim anyconnect.conf

server {

  listen 80;
   server_name 127.0.0.1 10.0.0.240;

   location / {
      #root /data/vpn;
      root /etc/anyconnect;
      autoindex on;
      index index.html;
   }

}
```
3.2 启动nginx

此时在浏览器就能访问到我创建的p12证书,这个证书稍后需要导入到客户端

```
[root@openvpn conf.d]$curl http://127.0.0.1/huangyong/
<html>
<head><title>Index of /huangyong/</title></head>
<body bgcolor="white">
<h1>Index of /huangyong/</h1><hr><pre><a href="../">../</a>
<a href="huangyong-cert.pem">huangyong-cert.pem</a>                                 11-Jul-2018 02:47                1176
<a href="huangyong-key.pem">huangyong-key.pem</a>                                  11-Jul-2018 02:47                5826
<a href="huangyong.p12">huangyong.p12</a>                                      11-Jul-2018 02:47                3376
<a href="user.tmpl">user.tmpl</a>                                          11-Jul-2018 02:47                 104
</pre><hr></body>
</html>
[root@openvpn conf.d]$
```
---

### 系统配置

#### 开启内核转发

```
vim /etc/sysctl.conf
net.ipv4.ip_forward = 1

编辑文件后,sysctl执行生效
sysctl -p
```

#### 开启防火墙转发

```
iptables -t nat -A POSTROUTING -s 10.0.81.0/24 -o eth0 -j MASQUERADE
```

---

### 测试

至此服务端配置完成.接下来测试一下IOS移动端连接情况(Android移动端操作类似)

1.前台开启ocserv服务

```
ocserv -c /etc/ocserv/ocserv.conf -f -d 1
```
2.使用手机登录VPN
关于手机使用VPN的操作手册详见另外一篇笔记.

---

### 密码认证方式

密码认证在服务端方面配置比较简单.但是在客户端日常使用方面可能每次登录VPN都需要输入用户密码就比较繁琐

> 据我测试用密码认证方式在MAC电脑上是可以的.但是证书认证方式有些问题


接下来讲解如何使用密码认证方式.

一.创建用户账户和密码.并且生产ocpasswd文件

```
[root@openvpn ocserv]$ocpasswd -c /etc/ocserv/ocpasswd caowei
Enter password:
Re-enter password:
```


二.编辑ocserv配置文件.主要修改以下内容:

```
[root@openvpn ocserv]$vim ocserv.conf

#注释证书认证方面的配置
#auth = "certificate"
#ca-cert = /etc/ocserv/ca-cert.pem

#开启密码认证.passwd指定ocpasswd文件路径
auth = "plain[passwd=/etc/ocserv/ocpasswd]"
```

其他方面配置和证书验证差不多.重启ocserv服务后,客户端就可以通过用户密码登录VPN

----


### 证书和密码认证同时使用.

ocserv在登录认证方面功能非常强大也很人性化.可以同时支持多种认证方式.

比如我们想要同时使用密码或者证书登录.


编辑配置文件:

```
#开启首选验证机制为密码认证

#auth = "pam"
#auth = "pam[gid-min=1000]"
auth = "plain[passwd=/etc/ocserv/ocpasswd]"
#auth = "certificate"
#auth = "radius[config=/etc/radiusclient/radiusclient.conf,groupconfig=true]"

# 开启证书备用认证"enable-auth"

# Specify alternative authentication methods that are sufficient
# for authentication. That is, if set, any of the methods enabled
# will be sufficient to login, irrespective of the main 'auth' entries.
# When multiple options are present, they are OR composed (any of them
# succeeding allows login).
enable-auth = "certificate"

#配置文件其他参数无需修改

```

重启ocserv服务后,客户端在没有证书的情况下会要求输入用户密码登录VPN.如果有导入证书的情况下,不会要求输入用户密码.

----

### 启动ocserv服务

没有任何问题后,可以开启ocserv服务

```
systemctl enable ocserv
systemctl start ocserv
```

---

### 客户端证书注销/账户

#### 删除一个账户.密码

```
#ocpasswd命令提供了delete选项删除用户

[root@openvpn anyconnect]$ocpasswd --help
ocpasswd - OpenConnect server password utility
Usage:  ocpasswd [ -<flag> [<val>] | --<name>[{=| }<val>] ]... [username]

   -c, --passwd=file          Password file
   -g, --groupname=str        User's group name
   -d, --delete               Delete user
   -l, --lock                 Lock user
   -u, --unlock               Unlock user
   -v, --version              output version information and exit
   -h, --help                 display extended usage information and exit

# 删除我的账户

[root@openvpn anyconnect]$ocpasswd -c /etc/ocserv/ocpasswd -d huangyong
```

#### 注销客户端证书

1.生成crl.tmpl模板文件

```
[root@openvpn anyconnect]$cat << _EOF_ >crl.tmpl
crl_next_update = 365
crl_number = 1
_EOF_

[root@openvpn anyconnect]$cat crl.tmpl
crl_next_update = 365
crl_number = 1
```

2.将要注销的证书文件拷贝一份到revoked.pem文件

```
[root@openvpn anyconnect]$cat huangyong/huangyong-cert.pem >> revoked.pem
```

3.生成crl.pem文件

```
certtool --generate-crl --load-ca-privkey ca-key.pem \
           --load-ca-certificate ca-cert.pem --load-certificate revoked.pem \
           --template crl.tmpl --outfile crl.pem
           
```

 例如:

 ```
[root@openvpn anyconnect]$certtool --generate-crl --load-ca-privkey ca-key.pem \
>            --load-ca-certificate ca-cert.pem --load-certificate revoked.pem \
>            --template crl.tmpl --outfile crl.pem
Generating a signed CRL...
Update times.

X.509 Certificate Revocation List Information:
	Version: 2
	Issuer: CN=VPN CA,O=DWD
	Update dates:
		Issued: Fri Jul 13 06:27:44 UTC 2018
		Next at: Sat Jul 13 06:27:44 UTC 2019
	Extensions:
		Authority Key Identifier (not critical):
			dcf4df0238da63a530982c3e40a83ad1432f01e9
		CRL Number (not critical): 01
	Revoked certificates (1):
		Serial Number (hex): 5b456fb4
		Revoked at: Fri Jul 13 06:27:44 UTC 2018
	Signature Algorithm: RSA-SHA256
	Signature:
		08:7b:a9:39:d1:63:b9:f6:7e:f0:6b:d3:d9:5f:c3:a4
		03:33:3b:1f:cf:33:83:54:23:b8:e0:e2:94:8c:d1:9a
		cb:82:48:5b:11:dc:7d:17:fe:12:44:8a:8f:db:a1:11
		73:a0:da:5d:55:1c:b3:9d:62:4f:b7:b9:c2:1a:d4:cd
		ba:c6:27:7b:c3:76:5d:9f:64:f8:6b:14:02:9a:3e:3a
		a4:e5:e7:14:f3:03:c5:68:f0:eb:bb:00:d5:c5:b9:dc
		83:1b:6d:2a:78:1c:f5:e0:88:3e:7b:f5:e0:23:0d:26
		f6:67:1e:8c:06:e5:02:50:2c:bc:17:71:5d:e0:0d:59
		3b:64:ab:82:9d:d1:01:ef:0f:96:05:cf:35:58:be:0f
		0d:b0:e2:40:33:11:e6:84:32:42:91:be:6b:0a:55:ad
		b4:29:88:3d:b9:aa:70:00:d9:02:d7:bd:f8:2b:65:ef
		4f:50:b0:fe:7d:d5:6c:9b:e8:70:0f:1e:2e:76:e9:d3
		cf:f3:e1:c1:8f:51:5c:8e:d7:f3:73:9b:5f:41:27:cc
		29:66:be:9c:a4:93:41:0c:15:95:da:52:26:e6:36:9b
		6e:09:52:02:a8:c9:df:4b:e3:98:98:2c:76:e4:1e:4a
		ce:3a:1b:f0:4f:bc:37:ad:54:63:ad:82:a9:70:50:35


 ```

4.修改配置文件

```
[root@openvpn anyconnect]$vim /etc/ocserv/ocserv.conf

#开启crl参数,并且制定crl.pem文件的路径
crl = /etc/anyconnect/crl.pem

```

5.重启ocserv服务

现在据我亲测证实无论我是用证书还是用我的账号密码都无法登陆VPN

----

