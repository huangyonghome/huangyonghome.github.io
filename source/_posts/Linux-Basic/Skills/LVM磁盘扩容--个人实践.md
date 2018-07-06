---
title:  Linux LVM磁盘扩容--个人实践
date: 2018-06-23 22:59:58
tags:  Linux,LVM
categories: [Linux-Basic,Skills ]
comments: true
copyright: true
---



## Linux LVM磁盘扩容--个人实践



#### 磁盘扩容可以分2种:

1.虚拟机原有磁盘上扩充

2.添加新磁盘,

> 这里讨论的是第一种情况. 

---

#### 实践背景:

虚拟机模板的磁盘为80G.在部署虚拟机的时候把磁盘修改为120G.(但是进入系统仍然是80G分区)

<!--more-->

---

#### 实践步骤:

1.fdisk /dev/sda命令新建一个逻辑分区,选择分区号(这里是3),并且转换为8e类型也就是lvm磁盘格式

分区完后.输入w保存分区修改.

partprobe重载分区.

reboot 重启(如果上一步没有生效)

{% qnimg linux/lvm.png %}

2.格式化分区:

```
mkfs -t ext4 /dev/sda3 
```

3.创建PV 

```
pvcreate /dev/sda3
```

{% qnimg linux/lvm1.png %}

4.查看当前VG信息 

```
vgdisplay
```

{% qnimg linux/lvm2.png %}

5.扩展vg 

```
vgextend VolGroup /dev/sda3 #将刚才创建的/dev/sda3的VG扩展到VolGroup的VG中
```

{% qnimg linux/lvm3.png %}

6.查看lvdisplay信息: 

{% qnimg linux/lvm4.png %}



7.扩展lv 

在这里我将新建的VG扩展到VolGroup-lv_root这个LV下.也就是根目录下

```
lvextend -L 89.9G /dev/mapper/VolGroup-lv_root  #这个命令表示讲lv_root这个lv扩充到89.9G(注意是扩展到)

还可以使用这个命令:
lvextend -L +39.9G /dev/mapper/VolGroup-lv_root  #这个命令表示将lv_root增加39.9G
(注意一个是增加到XXX.一个是增加XX)

或者使用如下命令:
lvextend -l +100%free /dev/mapper/VolGroup-lv_root  #这个命令表示分配所有可用容量,注意参数是小写l
```

扩充完可以看到lv_root这个lv已经从50G变成了89.9G

{% qnimg linux/lvm5.png %}



8.重设分区大小

```
resize2fs /dev/mapper/VolGroup-lv_root
```

9.查看是否成功扩容.

可见根目录从之前的50G扩展到89G.至此扩容成功.

{% qnimg linux/lvm6.png %}



