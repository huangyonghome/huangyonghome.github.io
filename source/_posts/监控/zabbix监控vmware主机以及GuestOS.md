---
title: zabbix监控vmware主机以及GuestOS
date: 2020-08-26 09:20:58
tags:  zabbix监控
categories: 监控
comments: true
copyright: true
---

### zabbix监控vmware主机以及GuestOS

#### 介绍

ESXI主机无法安装zabbix agent,所以不能使用传统的agent客户端监控vmware主机,但是Zabbix有自导的vmware hypervisors监控模板.Zabbix 通过 vmware collector 进程来监控虚拟机,使用SOAP协议从vmware web服务器获取必要的监控信息.

---

#### 准备工作

1.在zabbix服务器修改`zabbix_server.conf`配置文件

```
StartVMwareCollectors=6
VMwareCacheSize=50M
VMwareFrequency=10
VMwarePerfFrequency=60
VMwareTimeout=300
```

<!--more-->

**说明**: 

**StartVMwareCollectors**：vmware 收集器实例的数量。
此值取决于要监控的 VMware 服务的数量。在大多数情况下，这应该是：`servicenum < StartVMwareCollectors < (servicenum * 2)`其中 servicenum 是 VMware 服务的数量。

例如：如果您有 1 个 VMware 服务要将 StartVMwareCollectors 设置为 2，那么如果您有 3 个 VMware 服务，请将其设置为 5。请注意，在大多数情况下，此值不应小于 2，不应大于 VMware 数量的 2 倍服务。

2.重启zabbix服务

```
systemctl restart zabbix_server
```

---

#### Esxi物理主机配置

1.登陆Esxi web界面: https://172.16.0.55
2.在`manage`---`system`----`advanced settings`.修改`Config.HostAgent.plugins.solo.enableMob`的值为True

![image-20200818112727513](https://img2.jesse.top/image-20200818112727513.png)

3.访问:https://172.16.0.55/mob/?moid=ha-host&doPath=hardware.systemInfo
记录UUID
![image-20200818112949235](https://img2.jesse.top/image-20200818112949235.png)

4.在zabbix添加主机

![image-20200818113114835](https://img2.jesse.top/image-20200818113114835.png)

* **主机名称**:上面查到的UUID
* **IP地址**:Esxi的IP地址
* **端口**:80

**模板**:

![image-20200818113315388](https://img2.jesse.top/image-20200818113315388.png)

**宏**

![image-20200818113428859](https://img2.jesse.top/image-20200818113428859.png)

* **password**: Esxi主机密码
* **URL**: https://Esxi_IP/sdk 

* **username**: ESXI主机用户名
* **UUID**: 上文记录的UUID