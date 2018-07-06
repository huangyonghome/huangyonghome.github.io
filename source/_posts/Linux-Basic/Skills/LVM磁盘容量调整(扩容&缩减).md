---
title:  LVM磁盘容量调整(扩容&缩减)
date: 2018-06-23 22:59:58
tags:  Linux,LVM
categories: [Linux-Basic,Skills ]
comments: true
copyright: true
---

## LVM磁盘容量调整(扩容&缩减)


#### 背景:

今天安装一台新的虚拟机,Centos 7.4操作系统.使用的默认分区,但是默认将大部分
磁盘容量都分配给了Home分区:

```
[root@localhost ~]$df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root   50G  1.7G   49G   4% /
devtmpfs                 909M     0  909M   0% /dev
tmpfs                    920M     0  920M   0% /dev/shm
tmpfs                    920M  8.5M  912M   1% /run
tmpfs                    920M     0  920M   0% /sys/fs/cgroup
/dev/vda1               1014M  168M  847M  17% /boot
/dev/mapper/centos-home  447G   33M  447G   1% /home
tmpfs                    184M     0  184M   0% /run/user/0
```

这不符合需求,故此需要动态的将Home分区容量调整到根目录内.

<!--more-->

---

#### 操作步骤

**1.检查磁盘格式**

检查磁盘是ext4格式还是xfs格式.

```
[root@localhost ~]$cat /etc/fstab | grep centos-home
/dev/mapper/centos-home /home                   xfs     defaults        0 0
```

> Note:Centos7.4系统默认分区磁盘为xfs格式.xfs磁盘格式扩容不会对分区产生任何影响.
>
>但是**缩减需要重新格式化分区**,因此需要提前备份重要文件

**2.卸载home分区**

> 注意备份重要数据

```
[root@localhost ~]$umount /dev/centos/home
```

**3.将分区容量缩减到15G**

```
[root@localhost ~]$lvreduce -L 15G /dev/centos/home
  WARNING: Reducing active logical volume to 15.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce centos/home? [y/n]: y
  Size of logical volume centos/home changed from 446.99 GiB (114430 extents) to 15.00 GiB (3840 extents).
  Logical volume centos/home successfully resized.
```

**4.重新调整分区**

> Note.此时用resize2fs将不起作用,因为此命令只能对ext格式磁盘进行resize.

```
[root@localhost ~]$resize2fs /dev/centos/home
resize2fs 1.42.9 (28-Dec-2013)
resize2fs: Bad magic number in super-block while trying to open /dev/centos/home
Couldn't find valid filesystem superblock.
```
> xfs_growfs格式也会报错,此命令只能grow(扩容)磁盘,不能缩减.

```
[root@localhost ~]$xfs_growfs /dev/centos/home
xfs_growfs: /dev/centos/home is not a mounted XFS filesystem
```

> 直接挂载也会报错

```
[root@localhost ~]$mount /dev/centos/home /home
mount: /dev/mapper/centos-home: can't read superblock
```

故此,需要重新用mkfs.xfs重新格式化xfs磁盘

```
[root@localhost ~]$mkfs.xfs /dev/centos/home -f
meta-data=/dev/centos/home       isize=512    agcount=4, agsize=983040 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=3932160, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

**5.挂载分区**

```
[root@localhost ~]$mount /dev/centos/home /home
```

**5.重新调整根目录分区**

既然已经将home分区容量减少了.咱们就要把多余的容量增加到根分区

```
[root@localhost ~]$lvextend -l +100%FREE /dev/centos/root
  Size of logical volume centos/root changed from 50.00 GiB (12800 extents) to <482.00 GiB (123391 extents).
  Logical volume centos/root successfully resized.
```

**5.重新识别分区**

```
[root@localhost ~]$xfs_growfs /dev/mapper/centos-root
meta-data=/dev/mapper/centos-root isize=512    agcount=4, agsize=3276800 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0 spinodes=0
data     =                       bsize=4096   blocks=13107200, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal               bsize=4096   blocks=6400, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 13107200 to 126352384
```

**5.成功调整分区容量,检查结果**

```
[root@localhost ~]$df -h
Filesystem               Size  Used Avail Use% Mounted on
/dev/mapper/centos-root  482G  1.7G  481G   1% /
devtmpfs                 909M     0  909M   0% /dev
tmpfs                    920M     0  920M   0% /dev/shm
tmpfs                    920M  8.5M  912M   1% /run
tmpfs                    920M     0  920M   0% /sys/fs/cgroup
/dev/vda1               1014M  168M  847M  17% /boot
tmpfs                    184M     0  184M   0% /run/user/0
/dev/mapper/centos-home   15G   33M   15G   1% /home
```




