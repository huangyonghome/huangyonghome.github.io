---
title: Zabbix 监控Dell服务器硬件
date: 2020-06-26 09:20:58
tags:  zabbix监控
categories: 监控
comments: true
copyright: true
---



## Zabbix 监控Dell服务器硬件



### 介绍

​     OMSA（全称Openmanage Server Administrator),是戴尔公司自主研发的IT系统管理解决方案。其通过提供web的图形用户界面和操作系统的命令行工具对本地和远程的服务器进行管理和监控。

OMSA和IDRAC类似,产品,功能也相似.Zabbix可以通过OMSA和IDRAC来监控Dell服务器的硬件.

由于Dell服务器官网下载OMSA工具非常慢,所以安装很难.如果可以的花,首选还是建议通过IDRAC来监控服务器硬件状态

> 我在另一篇笔记<Zabbix 通过Dell Idrac监控服务器硬件状态>中介绍了如何通过IDRAC来监控Dell服务器硬件

<!--more-->

文章主要内容来源:https://www.jianshu.com/p/ecbd5e21924b

---



#### 1.在客户端服务器上安装Dell硬件监控工具-OMSA

```
#安装dell的yum源（8.2版本以上会出现问题）
wget -q -O - http://linux.dell.com/repo/hardware/DSU_15.12.00/bootstrap.cgi | bash
#最小化安装omsa
yum install -y srvadmin-base srvadmin-storageservices
#做软连接
ln -s /opt/dell/srvadmin/sbin/omreport /usr/bin/omreport
ln -s /opt/dell/srvadmin/sbin/omconfig /usr/bin/omconfig
#启动omsa
/etc/init.d/dataeng start
#加入到开机启动
chkconfig dataeng on
#清理yum源文件
rm -rf /etc/yum.repos.d/dell-*
```

> 由于Dell官网下载非常慢,所以yum安装非常慢.可以在Dell官网的服务器驱动程序下面页面下载OMSA管理工具手动安装,但是下载速度也非常慢



#### 2.在Zabbix客户端服务器上添加UserParameter配置

```
#下面是我的路径.(在zabbix-agentd.conf的Include配置中可以定义conf后缀文件路径)

vim /etc/zabbix/zabbix-agentd.d/userparameter_hardware.conf

#follow is monitor hardware
#状态1表示正常，状态0表示异常 
UserParameter=hardware_battery,omreport chassis batteries|awk '/^Status/{if($NF=="Ok") {print 1} else {print 0}}'
UserParameter=hardware_fan_health,awk -vhardware_fan_number=`omreport chassis fans|grep -c "^Index"` -vhardware_fan=`omreport chassis fans|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_fan_number==hardware_fan) {print 1} else {print 0}}'
UserParameter=hardware_memory_health,awk -vhardware_memory=`omreport chassis memory|awk '/^Health/{print $NF}'` 'BEGIN{if(hardware_memory=="Ok") {print 1} else {print 0}}'
UserParameter=hardware_nic_health,awk -vhardware_nic_number=`omreport chassis nics |grep -c "Slot"` -vhardware_nic=`omreport chassis nics |awk '/^Connection Status/{print $NF}'|wc -l` 'BEGIN{if(hardware_nic_number==hardware_nic) {print 1} else {print 0}}'
UserParameter=hardware_cpu,omreport chassis processors|awk '/^Health/{if($NF=="Ok") {print 1} else {print 0}}'
UserParameter=hardware_power_health,awk -vhardware_power_number=`omreport chassis pwrsupplies|grep -c "Index"` -vhardware_power=`omreport chassis pwrsupplies|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_power_number==hardware_power) {print 1} else {print 0}}'
UserParameter=hardware_temp,omreport chassis temps|awk '/^Status/{if($NF=="Ok") {print 1} else {print 0}}'|head -n 1
UserParameter=hardware_physics_health,awk -vhardware_physics_disk_number=`omreport storage pdisk controller=0|grep -c "^ID"` -vhardware_physics_disk=`omreport storage pdisk controller=0|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_physics_disk_number==hardware_physics_disk) {print 1} else {print 0}}'
UserParameter=hardware_virtual_health,awk -vhardware_virtual_disk_number=`omreport storage vdisk controller=0|grep -c "^ID"` -vhardware_virtual_disk=`omreport storage vdisk controller=0|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_virtual_disk_number==hardware_virtual_disk) {print 1} else {print 0}}'
```

> 原文中监控网卡是grep 关键字Interface Name.我改成了grep -c "Slot".这是因为Interface Name会grep到虚拟网卡.



### 3.重启zabbix agent

```
sudo systemctl restart zabbix-agent
```



#### 4. 在本地使用上面的命令测试

比如本地查看物理磁盘的状态是否都是oK

```
[work@bi-dev ~]$ awk -vhardware_physics_disk_number=`omreport storage pdisk controller=0|grep -c "^ID"` -vhardware_physics_disk=`omreport storage pdisk controller=0|awk '/^Status/{if($NF=="Ok") count+=1}END{print count}'` 'BEGIN{if(hardware_physics_disk_number==hardware_physics_disk) {print 1} else {print 0}}'
1
```

> 所有的监控项中,1代表正常,0代表有故障



#### 5.在zabbix服务端测试

命令:`zabbix_get -s 客户端IP -k 监控项`

```
#服务端的监控结果和客户端本地的执行结果一样
[work@server-4 ~]$ zabbix_get -s 172.16.10.212 -k hardware_physics_health
1
```



#### 6.配置主机,并且添加模板

模板已经fork了一份到我的github,地址是:[Monitor](https://github.com/huangyonghome/Monitor)

在这个仓库下的zabbix-Hardware目录下.有2个模板文件.

[Hardware-Check(2.4.6).xml](https://github.com/huangyonghome/Monitor/blob/master/zabbix-Hardware/Hardware-Check(2.4.6).xml) ---- 对应2.4.6版本的ZABBIX

[Hardware-Check(3.0.4).xml](https://github.com/huangyonghome/Monitor/blob/master/zabbix-Hardware/Hardware-Check(3.0.4).xml) ----对应3.0版本以上的Zabbix

