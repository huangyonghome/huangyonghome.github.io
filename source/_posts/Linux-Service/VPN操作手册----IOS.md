---
title: VPN操作手册----IOS
date: 2018-07-13 08:59:58
tags:  [Linux,anyconnect ]
categories: Linux-Service
comments: true
copyright: true
---



## VPN操作手册----IOS

### 简介

随着公司业务的不断上涨,规模不断扩大.数据信息也在快速增加.公司业务数据,特别是敏感用户数据的安全性也越来越重要.出于信息安全和数据安全考虑.技术部将采取一些列措施来防范,限制我们业务数据在互联网上的访问,确保数据安全性.

在公司内部搭建VPN通道就是安全性措施之一.公司内部数据将仅限于在公司内部访问.拒绝互联网上其他用户访问,破解公司数据的可能性.

<!--more-->

此后大家在公司外部(例如出差,在家办公等情况下)访问公司内部资源(例如,BI运营数据,服务器,数据库等)时,需要使用VPN软件.连接到公司的网络才能正常访问公司内部资源.

------

### VPN客户端选型

对于计算机(包括台式机,笔记本)我们使用的是Openvpn.因为openvpn比较稳定,使用非常广泛,配置使用方面简单,用户操作体验友好.

对于移动端(IOS,Andriod,IPAD).由于Openvpn的移动端客户端被禁.所以我们选择了思科的anyconnect VPN工具.

---

### 关于本教程

本教程旨在讲解如何在IOS设备使用公司VPN客户端软件..

本教程仅仅适用于苹果IOS设备(包括Iphone手机,IPAD平板电脑)

---

### 安装anyconnect VPN客户端

**1.下载VPN app**

在"app store"搜索并下载 "cisco anyconnect".如下图所示

![](https://img1.jesse.top/anyconnect1.png)





**2.阻止不信任服务器功能**

打开软件.点击底部的"设置"菜单.关闭"阻止不信任的服务器"功能.如下图所示:

![](https://img1.jesse.top/anyconnect2.png)

![](https://img1.jesse.top/anyconnect3.png)



**3.配置证书**

Anyconnect VPN支持包括密码,证书等多种验证方式.为了避免每次登陆VPN都需要输入一长串的用户名和密码,我们采取了证书认证方式.

3.1点击底部的"诊断"菜单.在诊断界面中,点击"证书"

![](https://img1.jesse.top/anyconnect4.png)



3.2.点击"导入用户证书......"

![](https://img1.jesse.top/anyconnect5.png)



3.3.输入证书所在的URL路径.这是你个人证书所在的服务器路径.

下图所示的是我个人的证书URL路径:

![](https://img1.jesse.top/anyconnect6.png)



3.4.证书导入成功

![](https://img1.jesse.top/anyconnect7.png)





**4.编辑VPN连接**

4.1 回到软件主界面,点击"连接".如下图所示

![](https://img1.jesse.top/anyconnect8.png)



4.2 点击"添加VPN连接".如下图所示

![](https://img1.jesse.top/anyconnect9.png)



4.3 按照下图所示编辑VPN服务器地址.编辑完成后点击右上角的"保存"按钮.如下图所示:

> "说明": 是指VPN连接的名字,可以任意输入连接名.
>
> "服务器地址": 这个是我们VPN服务器的地址.必须为:xx.xx.xx.xx:4333. **注意这里的冒号符号必须为英文格式,不要再中文输入法下输入冒号**

![](https://img1.jesse.top/anyconnect10.png)



4.4 此时会提示你"AnyConnect"想要添加VPN配置文件到iphone手机上.点击"Allow"允许.如下图所示:

![](https://img1.jesse.top/anyconnect15.png)



4.5.此时会要求你输入手机密码进行验证通过.如下图所示:

![](https://img1.jesse.top/anyconnect16.png)



4.6 在手机的设置界面,会看到VPN标识.另外在"设置"----"通用"界面也能看到VPN的配置.如下图所示:



![](https://img1.jesse.top/anyconnect17.png)



>  note: 由于我们使用的是证书方式登录.在这里的"设置"界面连接VPN的话识别不到证书,所以仍然需要使用anyconnect软件来连接VPN.

---



### 连接VPN到公司网络

上述的配置工作完成后,就可以连接VPN到公司内网了.连接的方法非常简单.

> **note:** 必须使用非公司网络才能连接VPN.比如4G运营商网络或者在公司外面的无线网络.因为您如果已经在公司内网里是没有办法也没有必要再通过VPN连接公司内网.**理解这一点非常重要**.



1.点击软件主界面的AnyConnect VPN按钮,启动VPN.如下图所示

> note: 注意"连接"这里是我们刚刚创建的dwd连接名.(有可能是您自己创建的连接名)

![](https://img1.jesse.top/anyconnect11.png)



2.此时会提示您服务器不受信任.由于我们的服务器证书是私签的免费证书,所以不被互联网第三方信任,

点击"继续"即可.如下图所示:

![](https://img1.jesse.top/anyconnect13.png)



3.连接成功后,按钮变成绿色,并且手机顶部会有"VPN"字样,.此时您就可以使用浏览器访问公司的内部网站了.



![](https://img1.jesse.top/anyconnect14.png)



4.如果您需要关闭VPN,直接点击"AnyConnect VPN" 绿色按钮关闭即可.

---
