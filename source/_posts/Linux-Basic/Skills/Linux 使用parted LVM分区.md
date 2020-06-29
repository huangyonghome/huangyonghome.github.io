---
title: Linux 使用parted+LVM分区
date: 2020-06-26 10:59:58
tags:  LVM
categories: [Linux-Basic,Skills ]
comments: true
copyright: true
---



## Linux 使用parted+LVM分区

#### 介绍

fdisk使用MBR分区硬盘,但是最大只能识别到2T的磁盘,如果单块磁盘超过2T则需要使用GPT分区格式

---

#### 分区步骤

分区方法和fdisk大同小异

* 1.parted 磁盘名 .进入交互式界面

```
[root@archiver-mysql ~]# parted /dev/vdb
GNU Parted 3.1
Using /dev/vdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
```

* 2.mklabel gpt 转换为gpt分区

```
(parted) mklabel gpt
Warning: The existing disk label on /dev/vdb will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? yes
```

3.mkpart分区

<!--more-->

```
(parted) mkpart
Partition name?  []? data  
File system type?  [ext2]? ext4
Start?
Start?
Start? 0% #起始和结束为止要用百分比格式,使4K对齐
End? 100%

#也可以直接在一条命令上输入
mkpart data ext4 0% 100% 
```

至此,gpt分区就完成了.可以看到单块磁盘有5T空间

```
(parted) print
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 5498GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  5498GB  5498GB               data

```

---

### 二 LVM分区

步骤一样

* 1.pvcreate /dev/vdb1

```
[root@archiver-mysql ~]# pvcreate /dev/vdb1
WARNING: ext4 signature detected on /dev/vdb1 at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/vdb1.
  Physical volume "/dev/vdb1" successfully created.
  
  [root@archiver-mysql ~]# pvdisplay
  "/dev/vdb1" is a new physical volume of "<5.00 TiB"
  --- NEW Physical volume ---
  PV Name               /dev/vdb1
  VG Name
  PV Size               <5.00 TiB
  Allocatable           NO
  PE Size               0
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               0SKSgB-TBHZ-PKU8-La1Q-dG00-23Df-4ZXoy4
```

* 2.vgextend vg名 pv名

```
[root@archiver-mysql ~]# vgdisplay
  --- Volume group ---
  VG Name               mysql_log_vggroup  #这个是vg名字
  System ID
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  2
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <100.00 GiB
  PE Size               4.00 MiB
  Total PE              25599
  Alloc PE / Size       25599 / <100.00 GiB
  Free  PE / Size       0 / 0
  VG UUID               JERXbu-vnc0-X2Kz-1PRH-2Q4v-kIc3-bRjNkE
  
  [root@archiver-mysql ~]# vgcreate mysql_data_vggroup /dev/vdb1 #这是vgcreate命令
  Volume group "mysql_data_vggroup" successfully created
```

* 3.lvcreate -n lv卷名字  -l +100%free vg卷名字

```
[root@archiver-mysql ~]# lvcreate -n mysql_data_lvgroup -l +100%free mysql_data_vggroup
```

* 4.格式化卷

```
[root@archiver-mysql ~]# mkfs.ext4 /dev/mysql_data_vggroup/mysql_data_lvgroup
```

* 5. 挂载

```
[root@archiver-mysql ~]# mount /dev/mysql_data_vggroup/mysql_data_lvgroup /data
```

* 6.别忘记写入/etc/fstab开启自启文件

```
echo "/dev/mapper/mysql_data_vggroup-mysql_data_lvgroup /data ext4 defaults 0 0" >> /etc/fstab
```



