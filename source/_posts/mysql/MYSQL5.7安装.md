---
title: MYSQL5.7安装
date: 2018-06-22 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---



## Centos7 yum源安装mysql 5.7



### 系统环境:

KVM服务器.

虚拟机版本:centos 7.4

IP地址:10.0.0.10

---



#### 安装步骤

<!--more-->

**一.下载mysql官方5.7版本的Yum源**

wget https://repo.mysql.com//mysql57-community-release-el7-11.noarch.rpm



**二.安装yum源**

```
yum localinstall mysql57-community-release-el7-11.noarch.rpm
```



>  Note:如果是安装最新版本(5.7),可以直接进行下一步操作.



 需要安装旧版本.可以修改配置Yum源:



**1.执行下列命令查看刚下载的Yum源支持的mysql版本**

```
shell> yum repolist all | grep mysql
```



2.如果是安装旧版本(比如安装5.6版本),可以关闭5.7版本的yum源.启用旧版本的yum源

```
shell> yum-config-manager --disable mysql57-community shell> yum-config-manager --enable mysql56-community
```



或者可以编辑yum源文件.修改enable字段来配置你想要安装的mysql版本.

```
enable=1表示开启源
enable=0表示关闭源
```



**三.安装MYSQL**

```
shell> yum install mysql-community-server 
```



**四.编辑mysql的配置文件.指定mysql的数据存储路径.(默认在/var/lib/mysql)**

在本场景下.mysql的数据文件路径是/data/mysql

```
sed -i 's@datadir=/var/lib/mysql@datadir=/data/mysql@g' /etc/my.cnf
```



**五.启动mysql服务**

```
systemctl start mysqld.service
```



> Note:mysql服务启动时,  
>
> 1.会自动初始化mysql数据库  
>
> 2.在mysql的数据文件目录下生成SSL证书文件,key文件  
>
> 3.生成初始密码



>  note:修改mysql的数据文件存储路径后,在启动mysql时报错:

```
2018-04-12T07:07:20.845077Z 0 [Note] /usr/sbin/mysqld (mysqld 5.7.21) starting as process 4442 ...
2018-04-12T07:07:20.847901Z 0 [Warning] Can't create test file /data/mysql/labs.lower-test
2018-04-12T07:07:20.847922Z 0 [Warning] Can't create test file /data/mysql/labs.lower-test
```



初步判断是数据文件路径没有权限.给数据文件路径添加权限:

```
chown -R mysql:mysql /data/mysql
chown -R mysql:mysql /var/run/mysqld
```

启动仍然报同样错误.关闭selinux.

```
sed -i 's@SELINUX=enforcing@SELINUX=disabled@g' /etc/selinux/config
```

重启服务器.mysql启动正常



**六.查看mysql服务状态**

```
systemctl status mysqld.service
```

**七.获取mysql的初始密码:**

```
[root@localhost ~]# grep 'temporary password' /var/log/mysqld.log
2018-04-10T14:24:09.501765Z 1 [Note] A temporary password is generated for root@localhost: d3jr!;9uJdyy

d3jr!;9uJdyy这个就是初始密码
```



**八.使用root用户密码登录mysql.修改用户密码.配置允许远程登录**

```
shell> mysql -uroot -p

修改密码:
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
```



> Note:密码要求至少一个大写字母,一个小写字母,一个数字,一个特殊符号总共8位字符



允许root用户远程登录:

```
mysql> update mysql.user set host='%' where user='root';
mysql> flush privileges;
```



**九.查看进程,端口是否正常启动**

```
netstat -tulnp | grep :3306
```

