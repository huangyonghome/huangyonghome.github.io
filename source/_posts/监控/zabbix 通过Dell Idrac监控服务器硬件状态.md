---
title: Zabbix 通过Dell Idrac监控服务器硬件状态群
date: 2020-06-26 09:20:58
tags:  zabbix监控
categories: 监控
comments: true
copyright: true
---

## Zabbix 通过Dell Idrac监控服务器硬件状态

### 背景

最近登陆vsphere控制台发现一台Dell服务器告警,已经损坏了.于是想通过zabbix监控Dell服务器的硬件状态.

有2种方法可以监控dell服务器的硬件状态

* 通过Dell的OMSA工具监控
* 通过Dell的Idrac模块的SNMP协议监控

Dell的OMSA工具需要安装在物理机系统上,我尝试过安装在vsphere的虚拟主机上,没办法获取到硬件状态.而且OMSA工具从Dell官网下载太慢.安装很麻烦

所以本文主要介绍如何通过idrac模块的snmp协议监控硬件状态.监控方法非常的简单

---

<!--more-->

### 1.登陆服务器的idrac平台.开启snmp协议(v2版本).

如果是idrac8版本,SNMP配置在左侧菜单栏`IDRAC设置`---`网络`----`服务`

如果是idrac9版本,SNMP配置在菜单栏`iDRAC设置`---`服务`---`SNMP代理`

> 根据我们的实际情况来看,无论是8还是9版本,SNMP都默认开启,端口是161,团体名是public



### 2.在zabbix服务端通过snmp命令检查是否能正确获取到服务器的snmp oid:

命令: `snmpwalk -v 2c -c 团体名 idrac IP地址`

```
[work@server-4 ~]$ snmpwalk -v 2c -c public 172.16.250.53
SNMPv2-MIB::sysDescr.0 = STRING:
SNMPv2-MIB::sysObjectID.0 = OID: SNMPv2-SMI::enterprises.674.10892.5
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (596160991) 69 days, 0:00:09.91
SNMPv2-MIB::sysContact.0 = STRING: \"support@dell.com\"
SNMPv2-MIB::sysName.0 = STRING: iDRAC-3D38Y23
SNMPv2-MIB::sysLocation.0 = STRING: \"unknown\"
SNMPv2-MIB::sysORLastChange.0 = Timeticks: (1) 0:00:00.01
SNMPv2-MIB::sysORID.1 = OID: SNMPv2-MIB::snmpMIB
SNMPv2-MIB::sysORID.2 = OID: SNMP-VIEW-BASED-ACM-MIB::vacmBasicGroup
SNMPv2-MIB::sysORID.3 = OID: TCP-MIB::tcpMIB
SNMPv2-MIB::sysORID.4 = OID: IP-MIB::ip
SNMPv2-MIB::sysORID.5 = OID: UDP-MIB::udpMIB
```



### 3. 去github上下载idrac snmp监控模板

地址: [idrac template](https://github.com/endersonmaia/zabbix-templates/tree/master/dell/idrac)

模板文件我已经fork到了我自己的github仓库:[zabbix-templates](https://github.com/huangyonghome/zabbix-templates)

**在dell/idrac下有3个文件:**

`ValueMaps_Dell_iDRAC.zbx.xml`-------导入键值对映射 (需要zabbix3.0版本以上)

`Template_Dell_iDRAC_SNMPv{23}.zbx.xml`-----监控模板(我们的idrac snmp协议是v2版本.所以采用snmpv2版本的模板)



### 4.配置全局变量

在Zabbix的`Administration`----`General`---右上角下拉框中选择宏Macros,新增宏定义:

```
#我这里Idrac的snmp团体名是public
{$SNMP_COMMUNITY_IDRAC} => public
```



### 5.导入模板

1.导入键值对映射`ValueMaps_Dell_iDRAC.zbx.xml`

2.导入监控模板`Template_Dell_iDRAC_SNMPv2.zbx.xml`



### 6.配置主机

主机名和随意定义.定义snmp接口的地址为idrac接口IP地址,端口默认161.

添加Template_Dell_iDRAC_SNMPv2模板

> 注意,snmp的地址是idrac地址,不是服务器iP地址

模板里没有配置图形,如果有需要可以自己添加.



### 7.验证

刚好这台服务器的物理磁盘有问题.可以看到Zabbix成功的监控到这个问题,并且发送给钉钉告警.

```
[故障]RAID Controller Error
告警级别：High
故障时间：2020.04.29 16:45:35
故障时长：1h 0m
IP地址：172.16.250.53
检测项：RAIDControllerStatus
*NonCritical (4) *
[exsi-idrac-53·故障 (31724858)]

[故障]Storage System Status Error
告警级别：Warning
故障时间：2020.04.29 16:45:35
故障时长：1h 0m
IP地址：172.16.250.53
检测项：GlobalSystemStorageStatus
*NonCritical (4) *
[exsi-idrac-53·故障 (31724857)]

[故障]Overall System Rollup Status Error
告警级别：Warning
故障时间：2020.04.29 16:45:35
故障时长：1h 0m
IP地址：172.16.250.53
检测项：GlobalSystemRollupStatus
*NonCritical (4) *
[exsi-idrac-53·故障 (31724855)]
```

服务器磁盘修复以后,监控回复正常.监控各项指标的结果不再是4(NonCritical) 而是3(OK)

```
work@server-4 ~]$ snmpwalk -v 2c -c public 172.16.250.53 1.3.6.1.4.1.674.10892.5.2.3.0
SNMPv2-SMI::enterprises.674.10892.5.2.3.0 = INTEGER: 3

[work@server-4 ~]$ snmpwalk -v 2c -c public 172.16.250.53 1.3.6.1.4.1.674.10892.5.5.1.20.130.1.1.37.1
SNMPv2-SMI::enterprises.674.10892.5.5.1.20.130.1.1.37.1 = INTEGER: 3
```

---

### 问题

目前发现一个模板的问题,在键值对映射中,Dell的OMSA(Open Manage System Status)键值对是这样映射的:

```
Dell Open Manage System Status
1  Other
2  Unknown
3  OK
4  NonCritical
5  Critical
6  NonRecoverable
```

在模板中,有一个触发器的名称是:Overall System Status Error

表达式为`{exsi-idrac-53:GlobalSystemStatus.last()}<>3`

用意是监控系统状态,如果值不为3(OK)则告警.但是根据实际情况来看,这个监控项GlobalSystemStatus.SNMP OID:1.3.6.1.4.1.674.10892.2.2.1.0正常情况下的值为0

```
[work@server-4 ~]$ snmpwalk -v 2c -c public 172.16.250.53 1.3.6.1.4.1.674.10892.2.2.1.0
SNMPv2-SMI::enterprises.674.10892.2.2.1.0 = INTEGER: 0
```

所以在模板中修改Overall System Status Error触发器的表达式为:

```
原表达式:
{Template Dell iDrac SNMPV2:GlobalSystemStatus.last()}<>3 

修改为:
{Template Dell iDrac SNMPV2:GlobalSystemStatus.last()}<>3 and 
{Template Dell iDrac SNMPV2:GlobalSystemStatus.last()}<>0
```

用意是这个监控项的结果既不等于3,又不等于0

