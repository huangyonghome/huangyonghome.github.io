---
title: OpenVpn 搭建教程
date: 2018-06-23 22:59:58
tags:  [Linux,openvpn ]
categories: Linux-Service
comments: true
copyright: true
---



## OpenVpn 搭建教程

参考下面文档  
http://www.startupcto.com/server-tech/centos/setting-up-openvpn-server-on-centos
--------

#### 环境:
KVM虚拟机  
操作系统:centos7.4  
openvpn版本:2.4  
easyrsa版本:3.0  
VPN客户端内网地址段:10.0.80.0/24  
公司服务器内网地址网段:10.0.0.0/24

<!--more-->


> note: RSA3.0的版本和2.0的使用有差别,注意区分


#### 步骤:

**一.获取新版本yum源**

```
[root@localhost ~]$wget  http://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-7-11.noarch.rpm
```

**二.安装yum源**

```
[root@localhost ~]$rpm -Uvh epel-release-7-11.noarch.rpm
```

**三.安装openvpn和easy-rsa**

```
[root@localhost ~]$ yum install easy-rsa openssh-server lzo openssl openssl-devel openvpn NetworkManager-openvpn openvpn-auth-ldap
```

**四.拷贝server.conf配置文件到/etc/openvpn**

```
[root@localhost ~]$cp /usr/share/doc/openvpn-2.4.6/sample/sample-config-files/server.conf /etc/openvpn
```

**五.拷贝easy-rsa程序到/etc/openvpn**

```
[root@localhost openvpn]$cp -R /usr/share/easy-rsa/ /etc/openvpn
[root@localhost openvpn]$cd /etc/openvpn/easy-rsa/3
```

**六.easyrsa初始化私钥..easyrsa3.0的配置步骤和老版本2.0比有点变化.不需要定义var变量.**

```
[root@localhost 3.0.3]$./easyrsa init-pki
```
```
[root@localhost 3.0.3]$./easyrsa build-ca nopass
Generating a 2048 bit RSA private key
...+++
......................+++
writing new private key to '/etc/openvpn/easy-rsa/3.0.3/pki/private/ca.key.4VoxQD3h0Y'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Common Name (eg: your user, host, or server name) [Easy-RSA CA]:dwd

CA creation complete and you may now import and sign cert requests.
Your new CA certificate file for publishing is at:
/etc/openvpn/easy-rsa/3.0.3/pki/ca.crt
```

**七.创建服务端秘钥.openvpn表示openvpn服务器的服务器名称.nopass选项说明不需要密码.**

```
[root@localhost 3.0.3]$./easyrsa build-server-full openvpn nopass
Generating a 2048 bit RSA private key
...+++
...........................................................................................+++
writing new private key to '/etc/openvpn/easy-rsa/3.0.3/pki/private/openvpn.key.XpsmN8YpAx'
-----
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'openvpn'
Certificate is to be certified until May 28 08:57:19 2028 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
```


**八.生成dh密码算法**

```
[root@localhost 3.0.3]$./easyrsa gen-dh
```

**九.拷贝生成的秘钥到openvpn的配置文件夹下**

```
[root@localhost 3.0.3]$cd pki
[root@localhost pki]$ls
ca.crt  certs_by_serial  dh.pem  index.txt  index.txt.attr  index.txt.attr.old  index.txt.old  issued  private  reqs  serial  serial.old
[root@localhost pki]$cp dh.pem ca.crt /etc/openvpn/
[root@localhost pki]$cp issued/openvpn.crt /etc/openvpn/server.crt
[root@localhost pki]$cp private/openvpn.key /etc/openvpn/server.key
```

**十.修改server.conf配置文件**

> 关于配置文件的说明可以参考:https://my.oschina.net/liucao/blog/863112

```
[root@openvpn ~]$sed -e 's/^[;#].*//g' /etc/openvpn/server.conf | sed '/^$/d'
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key  # This file should be kept secret
dh dh.pem
server 10.0.80.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 10.0.0.0 255.255.0.0"
push "redirect-gateway"
push "dhcp-option DNS 114.114.114.114"
push "dhcp-option DNS 210.22.70.3"
duplicate-cn
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
log-append  openvpn.log
verb 3
explicit-exit-notify 1
```
> 特别注意push "redirect-gateway"选项和duplicate-cn选项..下面来重点分析一下
>
> 1.**redirect-gateway**参数表示将推送一条指向本机的默认网关到客户端.这就意味着只要客户端拨了VPN,不管他访问什么,只要是访问互联网就走VPN通道.先到VPN服务器,由你VPN服务器转发请求再返回给你客户端.这样一来访问互联网速度会受很大影响.
>
> 注释该参数,表示不推送默认网关给客户端.这样客户端只访问公司内网服务器才走VPN通道,而访问互联网仍然是走本地网络.在我这个例子中.push "route 10.0.0.0 255.255.0.0"表示推送给客户端的公司内网服务器网段.当客户端访问10.0.0.0/16网络时走VPN通道
>
> 2.**duplicate-cn:**这个选项表示可以多人同时共用一个客户端账号.如果注释此选项,则同一个账号,同时只能一个人使用.如果有第二个人用同样的账户登录,会将之前的使用该账号登录的人挤下去.这一点后续还会讲到



**十一.关闭centos7自带的firewalld.然后保存iptables设置,开启自起**

```
[root@openvpn ~]$systemctl stop firewalld.service
[root@openvpn ~]$systemctl disable firewalld.service
[root@openvpn ~]$yum install iptables-services
[root@openvpn ~]$systemctl enable iptables.service
[root@openvpn ~]$systemctl start iptables
[root@openvpn ~]$service iptables save
```

**十二.修改防火墙.将下列保存成脚本文件,然后执行**

```
#!/bin/bash

# REMEMBER: Run this as a single bash script or you'll lock yourself out of your machine.

# Flushing all rules
iptables -F FORWARD
iptables -F INPUT
iptables -F OUTPUT
iptables -X
# Setting default filter policy
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP
# Allow unlimited traffic on loopback
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
# Accept outbound on the primary interface
iptables -I OUTPUT -o eth0 -d 0.0.0.0/0 -j ACCEPT
# Accept inbound TCP packets
iptables -I INPUT -i eth0 -m state --state ESTABLISHED,RELATED -j ACCEPT
# Allow incoming SSH
iptables -A INPUT -p tcp --dport 22 -m state --state NEW -s 0.0.0.0/0 -j ACCEPT
# Allow incoming OpenVPN
iptables -A INPUT -p udp --dport 1194 -m state --state NEW -s 0.0.0.0/0 -j ACCEPT
# Enable NAT for the VPN
iptables -t nat -A POSTROUTING -s 10.0.80.0/24 -o eth0 -j MASQUERADE
# Allow TUN interface connections to OpenVPN server
iptables -A INPUT -i tun0 -j ACCEPT
# Allow TUN interface connections to be forwarded through other interfaces
iptables -A FORWARD -i tun0 -j ACCEPT
iptables -A OUTPUT -o tun0 -j ACCEPT
iptables -A FORWARD -i tun0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i eth0 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT
# Allow outbound access to all networks on the Internet from the VPN
iptables -A FORWARD -i tun0 -s 10.0.80.0/24 -d 0.0.0.0/0 -j ACCEPT
# Block client-to-client routing on the VPN
iptables -A FORWARD -i tun0 -s 10.0.80.0/24 -d 10.0.80.0/24 -j DROP
```
> 上面的Iptables规则过于严格.如果你的服务器Iptables不需要做任何限制.那么只需要添加一条iptables的nat规则就可以了:
```
iptables -t nat -A POSTROUTING -s 10.0.80.0/24 -o eth0 -j MASQUERADE
```

**十三.开启OpenVPN服务器的网卡转发功能***

```
echo "net.ipv4.ip_forward = 1" >>/etc/sysctl.conf
sysctl -p
```

**十四.启动openvpn**

```
# sudo systemctl -f enable openvpn@server.service
# sudo systemctl start openvpn@server.service
```
>note: 这个openvpn@server格式的server表示Openvpn使用erver.conf配置文件启动


**十五.创建一个客户端秘钥.dwdtech是客户端名称,nopass选项表示不需要密码**

```
[root@localhost 3.0.3]$./easyrsa build-client-full dwdtech nopass
Generating a 2048 bit RSA private key
......................................+++
.........................................................................................................................................+++
writing new private key to '/etc/openvpn/easy-rsa/3.0.3/pki/private/dwdtech.key.klfqyxWXig'
-----
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'dwdtech'
Certificate is to be certified until May 28 09:03:05 2028 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
```

**十六.配置客户端的配置文件**

1.创建客户端配置文件夹.拷贝客户端秘钥

```
[root@openvpn 3]$ mkdir ~/vpn-client
[root@openvpn 3]$ cd /etc/openvpn/easy-rsa/3

[root@openvpn 3]$cp pki/ca.crt ~/vpn-client/
[root@openvpn 3]$cp pki/issued/dwdtech.crt ~/vpn-client/client.crt
[root@openvpn 3]$cp pki/private/dwdtech.key ~/vpn-client/client.key
```

2.创建客户端配置文件

```
[root@openvpn 3]$cd ~/vpn-client/

[root@openvpn vpn-client]$vim client.ovpn

client
dev tun
proto udp
remote xx.xx.xx.xx 1194  #客户端远程拨号公司出口公网IP地址
resolv-retry infinite
redirect-gateway
nobind
persist-key
persist-tun
ca ca.crt
cert client.crt
key client.key
comp-lzo
verb 3
sndbuf 393216  #关于sndbuf和rcvbuf参数,请参考另一篇笔记
rcvbuf 393216

[root@openvpn vpn-client]$ls
ca.crt  client.crt  client.key  client.ovpn
```

3.打包配置文件.下载到本地,并且拷贝到客户端

```
[root@openvpn ~]$tar -cvf vpnclient.tar vpn-client
vpn-client/
vpn-client/ca.crt
vpn-client/client.crt
vpn-client/client.key
vpn-client/client.ovpn
vpn-client/vpnclient.tar

```
十七.修改出口设备的NAT转发

在出口设备上增加一条NAT转发规则.注意是UDP协议转发.转发1194端口到Openvpn服务器的1194端口

----------------------------------------------------------------------------

至此配置完成..接下来使用openvpn客户端拨号

推荐VPN客户端软件 Windows: OpenVPN && Mac：Tunnelblick

如果是windows,安装完以后把服务器上配置的客户端秘钥,配置文件拷贝到openvpn安装路径的config路径下

如果是MAC,安装完成后把ovpn配置文件拖拽到Tunnelblick软件界面即可

---


## 第二部分: 多账号使用Openvpn


一个比较懒散的做法是像刚才的教程一样,所有人都使用同一个秘钥账号连接Openvpn.但是公司里如果所有人都使用一个秘钥的话,人员离职后,仍然可以使用该秘钥通过Openvpn访问公司内网,非常不安全.  
而此时重新创建秘钥则又"前一发动全身",意味着每个人都要更新VPN秘钥.非常复杂.

此时就需要对于每个用户创建并注销单独的秘钥.接下来的教程演示如何实现这一功能

---

#### 一.创建多个客户端账户

创建客户端账户步骤和刚才教程一模一样


**1.修改server.conf配置文件,注释duplicate-cn参数**

```
[root@openvpn openvpn]$vim server.conf

# Uncomment this directive if multiple clients
# might connect with the same certificate/key
# files or common names.  This is recommended
# only for testing purposes.  For production use,
# each client should have its own certificate/key
# pair.
#
# IF YOU HAVE NOT GENERATED INDIVIDUAL
# CERTIFICATE/KEY PAIRS FOR EACH CLIENT,
# EACH HAVING ITS OWN UNIQUE "COMMON NAME",
# UNCOMMENT THIS LINE OUT.
;duplicate-cn
```

> Note: 开启了这个参数就表示每个客户端秘钥账户就只能同时一个人使用,如果多个人同时同一个账户会挤掉前面已经连接上的vpn的用户:

下面是故障出现时,服务器Openvpn日志的报错:

```
Sat Jun 16 01:08:56 2018 10.0.99.1:29167 [dwdtech] Peer Connection Initiated with 

Sat Jun 16 01:08:56 2018 MULTI: new connection by client 'dwdtech' will cause previous active sessions by this client to be dropped.  Remember to use the --duplicate-cn option if you want multiple clients using the same certificate or username to concurrently connect.
```


**2.创建另外一个客户端秘钥**

```
[root@openvpn 3]$./easyrsa build-client-full test nopass
Generating a 2048 bit RSA private key
..........+++
....................+++
writing new private key to '/etc/openvpn/easy-rsa/3/pki/private/test.key.VGKwuGH2p6'
-----
Using configuration from ./openssl-1.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
commonName            :ASN.1 12:'test'
Certificate is to be certified until Jun 12 06:33:21 2028 GMT (3650 days)

Write out database with 1 new entries
Data Base Updated
```

此时就创建了多个客户端秘钥

---

#### 二.注销客户端账户

这里以刚才的test账户为例

**1.删除该客户端秘钥**


```
[root@openvpn 3]$./easyrsa revoke test


Please confirm you wish to revoke the certificate with the following subject:

subject=
    commonName                = test


Type the word 'yes' to continue, or any other input to abort.
  Continue with revocation: yes
Using configuration from ./openssl-1.0.cnf
Revoking Certificate D6A90A078F182F97CBB3C37172DFFFC7.
Data Base Updated

IMPORTANT!!!

Revocation was successful. You must run gen-crl and upload a CRL to your
infrastructure in order to prevent the revoked cert from being accepted.
```

**2.按照提示,生成crl文件**

```
[root@openvpn 3]$./easyrsa gen-crl
Using configuration from ./openssl-1.0.cnf

An updated CRL has been created.
CRL file: /etc/openvpn/easy-rsa/3/pki/crl.pem
```

**3.复制crl.pem文件到OpenVpn根目录下**

```
[root@openvpn 3]$cp pki/crl.pem /etc/openvpn/
```

**4.添加配置到server.conf配置文件,开启crl验证功能,并且制定crl文件路径.crl文件可以用绝对路径也可以用相对路径**

添加下面这一行到server.conf配置文件

```
[root@openvpn openvpn]$tail -1 server.conf
crl-verify /etc/openvpn/crl.pem
```
使用下面命令可以检查crl.pem文件查看被注销的账户信息

```
[root@openvpn openvpn]$openssl crl -in /etc/openvpn/crl.pem -text -noout
Certificate Revocation List (CRL):
        Version 2 (0x1)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: /CN=dwd
        Last Update: Jun 15 06:53:38 2018 GMT
        Next Update: Dec 12 06:53:38 2018 GMT
        CRL extensions:
            X509v3 Authority Key Identifier:
                keyid:87:B0:CC:51:14:B2:0D:C3:75:75:D1:9C:AB:0D:2E:3C:D8:F7:05:8D
                DirName:/CN=dwd
                serial:8F:2C:45:27:E1:42:46:D3

Revoked Certificates:
    Serial Number: D6A90A078F182F97CBB3C37172DFFFC7
        Revocation Date: Jun 15 06:53:07 2018 GMT
    Signature Algorithm: sha256WithRSAEncryption
         be:59:e5:14:06:a9:5c:cd:41:60:1c:e2:81:db:a0:47:2c:75:
         9d:0f:75:14:6b:fc:ff:f2:3f:51:8b:f6:c5:d4:cf:1b:13:41:
         24:2f:94:fb:29:e9:a9:2d:fd:5f:b3:c0:af:42:95:65:a4:00:
         44:55:bd:b4:61:26:64:4b:d0:51:49:02:94:cd:d0:71:98:99:
         5d:2a:a1:b4:a1:01:2e:9c:2e:dc:f9:44:5f:23:c9:6c:56:47:
         df:35:e2:9f:05:3d:98:6b:80:61:ce:36:be:01:df:18:22:36:
         c6:fe:14:bf:aa:55:de:2b:ca:9c:17:03:49:60:47:0f:7d:e2:
         f4:c2:75:41:9e:54:88:49:ce:29:c6:4d:79:db:68:ed:46:64:
         3f:98:19:c7:72:73:c1:1a:69:03:9e:a2:57:43:04:61:fa:94:
         43:18:24:fc:5b:3b:62:e6:4e:5e:be:28:b5:dd:ea:16:cc:47:
         2e:62:86:15:cb:ff:4f:1a:a1:e6:40:9f:7e:11:00:3b:b4:41:
         1b:1a:13:cd:24:ce:83:34:88:2f:c8:05:b3:0f:af:f7:6a:c8:
         be:71:08:92:e5:26:64:ae:7b:92:52:b3:3c:02:3d:cd:1a:d7:
         bd:f2:7a:93:02:73:5f:7e:3a:94:74:ab:ff:49:1e:d1:b0:f9:
         c3:88:b5:6c
[root@openvpn openvpn]$
```

在pki目录下.有一个index.txt文件.此文件列举了活跃的和已经注销的客户端证书信息:

```
cd /etc/openvpn/easy-rsa/3/pki

[root@openvpn pki]$cat index.txt
V	280528085719Z		9BF6331DD9E82F354F8E5867591E15BC	unknown	/CN=openvpn
V	280528090305Z		AC3AB23DB02135A37CFB20B158813BF1	unknown	/CN=dwdtech
R	280612063321Z	180615065307Z	D6A90A078F182F97CBB3C37172DFFFC7	unknown	/CN=test
V	280612070037Z		E39E66223879C19CB521589A8F82246B	unknown	/CN=yongge
R	280629065514Z	180703015608Z	871C32C7042E841B625C34E57D526039	unknown	/CN=huangyong
```
> 第一个字母R表示已经注销的账户.

此时,test客户端虽然有秘钥,但是无法拨入Openvpn.服务端的openvpn.log日志记录了这一行为:

```
Tue Jul  3 02:23:47 2018 10.0.99.1:23273 VERIFY ERROR: depth=0, error=certificate revoked: CN=test
Tue Jul  3 02:23:47 2018 10.0.99.1:23273 OpenSSL: error:14089086:SSL routines:ssl3_get_client_certificate:certificate verify failed
Tue Jul  3 02:23:47 2018 10.0.99.1:23273 TLS_ERROR: BIO read tls_read_plaintext error
Tue Jul  3 02:23:47 2018 10.0.99.1:23273 TLS Error: TLS object -> incoming plaintext read error
Tue Jul  3 02:23:47 2018 10.0.99.1:23273 TLS Error: TLS handshake failed
Tue Jul  3 02:23:47 2018 10.0.99.1:23273 SIGUSR1[soft,tls-error] received, client-instance restarting
```
后续删除其他客户端账户,需要重复上面1-3步骤.

>note:我在这里踩过很深的坑.如果需要注销多个账户时,会生成新的crl.pem文件.此时直接把新的crl.pem文件覆盖老的就可以,其他不用管.
请看我的stackoverflow的提问:[how to revoke multiple OpenVPN clients Certificate](https://stackoverflow.com/questions/51146190/how-to-revoke-multiple-openvpn-clients-certificate)

---


### 第三部分.Openvpn加速

这篇文档详细解释了为什么Openvpn拨号之后宽带速度很慢的原因  
https://www.lowendtalk.com/discussion/40099/why-openvpn-is-so-slow-cool-story

#### 配置步骤:

**1.修改内核缓冲区参数**

```
[root@openvpn ~]$vim /etc/sysctl.conf

net.core.rmem_default = 393216
net.core.wmem_default = 393216

[root@openvpn ~]$sysctl -p
net.ipv4.ip_forward = 1
net.core.rmem_default = 393216
net.core.wmem_default = 393216
```

**2.修改Openvpn的配置文件,添加下面2个参数**

```
[root@openvpn ~]$tail -2 /etc/openvpn/server.conf
sndbuf 393216
rcvbuf 393216

```
> 如果客户端文件已经下发给用户,无法手动修改,则继续在server.conf添加下面2个参数,将配置推送给客户端  
push "sndbuf 393216"  
push "rcvbuf 393216"

**3.重启openvpn**

```
[root@openvpn openvpn]$systemctl restart  openvpn@server.service
```

**4.重新编辑客户端的ovpn配置文件,添加下面2个参数**

```
[root@openvpn ~]$tail -2 vpn-client/client.ovpn
sndbuf 393216
rcvbuf 393216
```
**5.将客户端配置文件重新下载到本地,拷贝到openvpn安装目录的conf目录下.重新拨号**

---

### 第四部分 批量创建openvpn客户端账号

每次手动创建客户端账号的工作太繁琐.于是写了一个脚本来进行批量化创建工作

**脚本功能如下:**

1.使用-f参数从文件中批量读取账户进行创建,也可以使用-u参数指定要创建的账户.

2.判断账户是否已经存在

3.创建完客户端证书和秘钥文件后,会自动创建客户端配置文件.

4.打包客户端所需要的文件到/root目录下.

>  如果需要加入注销客户端账号的功能可以自行修改脚本,或者联系本人

```
#!/bin/bash
#description:批量创建Openvpn的用户账户.可以指定创建单个vpn账号,也可以创建一个账号文件.批量从文件中读取账号来创建
#author:huangyong
#date:2018-07-02


#创建账户函数
function create_user () {

name=$1

vpnDir="/etc/openvpn/easy-rsa/3"
homeDir="/root/${name}"
cert="${name}.crt"
key="${name}.key"

# 判断账户是否已经创建过了,如果是,则直接返回到主程序
[ -f $vpnDir/pki/issued/${cert} ] && echo "${name} is already existed " && return 0

#进入到easyrsa目录下创建不带密码的VPN账户.这里必须要在想对目录内执行easyrsa命令
cd $vpnDir
./easyrsa build-client-full ${name} nopass > /dev/null 2>&1

#如果创建账户失败,则退出整个程序
[ $? -ne 0 ] && exit 1 


#将创建的账户证书,秘钥文件拷贝到该用户的文件夹内
[ -d $homeDir ] || mkdir $homeDir
cp $vpnDir/pki/ca.crt $homeDir
cp $vpnDir/pki/issued/${cert} $homeDir
cp $vpnDir/pki/private/${key} $homeDir

# 创建客户端账户配置文件
cat > $homeDir/${name}.ovpn << EOF
client
dev tun
proto udp
remote xx.xx.xx.xx 1194  #客户端远程拨号公司出口公网IP地址
resolv-retry infinite
redirect-gateway
nobind
persist-key
persist-tun
ca ca.crt
cert $cert
key $key
comp-lzo
verb 3

EOF

#打包账户文件
tar -C /root -cf ${homeDir}.tar ${homeDir} > /dev/null 2>&1

#删除账户文件夹
rm -rf ${homeDir}

}


# 检测用户是否输入了正确的选项

if ! ` echo $1 | grep '-' > /etc/null 2>&1 `;then
    echo "Usage $0 {-f filename contains a group of account name | -u specify an account }"
fi

# 读取用户输入的选项和参数
while getopts "f:u:" SWITCH;do

   case $SWITCH in
         
        f)
          #如果用户指定了一个账户文件,则判断用户输入的账号文件是否存在
          [ ! -f $OPTARG ] && echo " the file doesn't exist" && exit 1

          #读取文件,循环获取文件内的账号
          cat $OPTARG | while read line
             do
                 create_user $line  #调用函数
                 [ $? -ne 0 ] && echo " something wrong" && exit 1 || echo "create ${line} successfully"
          done
          ;;

       u)
         #将用户输入的用户名代入到函数
         create_user $OPTARG
         [ $? -ne 0 ] && echo " something wrong" && exit 1 || echo " create $OPTARG successfully"
         ;;
       
       *)
        echo "Usage $0 {-f filename contains a group of account name | -u specify an account }"
        ;;
  esac
done
```

例如从文件中批量创建账号:

```
[root@openvpn ~]$./createAccount.bak.sh -f name.txt
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
create VPN account successfully
[root@openvpn ~]$
```








