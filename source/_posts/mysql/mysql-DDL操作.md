---
title: DDL操作命令
date: 2018-06-22 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---


## DDL操作命令

#### 简介

DDL (Data Definition Language) ----数据定义语言.定义数据库,表,列,索引,字段等数据库对象,常用的语句有:create,drop,alter

---

#### 一.数据库操作

<!--more-->

1.创建一个数据库:

```
CREATE DATABASE dbname
mysql> create database mytest;
Query OK, 1 row affected (0.00 sec)
```

2.查看当前数据库中存在哪些数据库

```
show databases
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| mytest             |
| performance_schema |
| test               |
| test1              |
+--------------------+
6 rows in set (0.00 sec)
```

> information_schema:主要存储系统中一些数据库对象信息.比如用户表信息,列信息,权限,字符集,分区等  
>cluster: 存储系统集群信息  
>mysql:存储系统用户权限  

3.使用数据库

```
use mytest
mysql> use mytest;
Database changed
```

4.查看刚才使用的数据库中的所有表

```
show tables
mysql> show tables;
Empty set (0.00 sec)
```

>这是刚创建的数据库,所以没有任何表

5.删除一个数据库

```
drop database dbname;
mysql> drop database test1;
Query OK, 0 rows affected (0.00 sec)
```

>note:请谨慎操作任何drop,delete等删除相关语句

--------------------------------------------------------------------------------

#### 二.数据表操作

1.在数据库中创建一张表

```
CREATE TABLE 表名(列名1,字段类型,约束条件
                                列名2,字段类型,约束条件
                                列名n,字段类型,约束条件)

```


列如:创建一个emp表,包含ename(列名1,字段类型是varchar(10)).hiredate(列名2,字段类型是date),sal(列名3,字段类型int(2),以及deptno

```
mysql> create table emp(ename varchar(10),hiredate date,sal int(2),deptno int(2));
Query OK, 0 rows affected (0.04 sec)
```

2.查看表的定义:

语法:

```
DESC 表名;
```

```
mysql> desc emp;
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| ename    | varchar(20) | YES  |     | NULL    |       |
| hiredate | date        | YES  |     | NULL    |       |
| sal      | int(2)      | YES  |     | NULL    |       |
| deptno   | int(2)      | YES  |     | NULL    |       |
+----------+-------------+------+-----+---------+-------+
4 rows in set (0.00 sec)
```

3.查看创建该表事的SQL语句:

语法:

```
show create table 表名;
```

例如.查看emp表是如何创建的

```
mysql> show create table emp;
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Table | Create Table |
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| emp | CREATE TABLE `emp` (
`ename` varchar(10) DEFAULT NULL,
`hiredate` date DEFAULT NULL,
`sal` int(2) DEFAULT NULL,
`deptno` int(2) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1 ###除了可以看到表定义外,还看到了数据库引擎和字符集
+-------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```


4.删除表

语法:

```
DROP TABLE 表名
```

例如.删除emp表

```
drop table emp;
```

5.修改表----alter table语句


5.1修改表类型

语法:

```
ALTER TABLE 表名 MODIFY 列名 列字段定义 [FIRST | AFTER 列名]
```

例如:将emp表中的ename列的字段类型从varchar(10) 改成varchar(20)

```
mysql> alter table emp modify ename varchar(20);
Query OK, 0 rows affected (0.09 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> desc emp;
+----------+-------------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| ename | varchar(20) | YES | | NULL | |
```


5.2增加表字段

语法

```
ALTER TABLE 表名 ADD 列名 列字段类型 [FIRST | AFTER 列名]
```

例如:增加age列,字段类型为int(3)

```
mysql> alter table emp add age int(3);
Query OK, 0 rows affected (0.07 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> desc emp;
+----------+-------------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| ename | varchar(20) | YES | | NULL | |
| hiredate | date | YES | | NULL | |
| sal | int(2) | YES | | NULL | |
| deptno | int(2) | YES | | NULL | |
| age | int(3) | YES | | NULL | |
+----------+-------------+------+-----+---------+-------+
5 rows in set (0.00 sec)
```


5.3 删除表字段

语法:

```
ALTER TABLE 表名 DROP 列名
```

例如:将age删掉

```
mysql> alter table emp drop age;
Query OK, 0 rows affected (0.03 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> desc emp;
+----------+-------------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| ename | varchar(20) | YES | | NULL | |
| hiredate | date | YES | | NULL | |
| sal | int(2) | YES | | NULL | |
| deptno | int(2) | YES | | NULL | |
+----------+-------------+------+-----+---------+-------+
4 rows in set (0.00 sec)
```

5.4 字段改名.

语法:

```
ALTER TABLE 表名 CHANGE 旧列名 新列名 列字段类型
```

例如:将age改为age1,同时将字段类型改为int(3)

```
mysql> alter table emp change age age1 int(4);
Query OK, 0 rows affected (0.01 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> desc emp;
+----------+-------------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| ename | varchar(20) | YES | | NULL | |
| hiredate | date | YES | | NULL | |
| sal | int(2) | YES | | NULL | |
| deptno | int(2) | YES | | NULL | |
| age1 | int(4) | YES | | NULL | |
+----------+-------------+------+-----+---------+-------+
5 rows in set (0.01 sec)
```

> Note:change和notify都可以修改表定义.change需要些两次列名.但是change可以修改队列名,modify不可以

>Note:前面介绍的字段增加和修改语法中都有一个[FIRST|AFTER 列名]的可选项.这个选项用来修改某个列名在表中的位置.ADD默认将列放在表的最后位置,CHANGE/MODIFY不会修改表位置.


例如:将age1列名改回到age,并且移动到ename列之后

```
mysql> alter table emp change age1 age int(4) after ename;
Query OK, 0 rows affected (0.07 sec)
Records: 0 Duplicates: 0 Warnings: 0

mysql> desc emp;
+----------+-------------+------+-----+---------+-------+
| Field | Type | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| ename | varchar(20) | YES | | NULL | |
| age | int(4) | YES | | NULL | |
| hiredate | date | YES | | NULL | |
| sal | int(2) | YES | | NULL | |
| deptno | int(2) | YES | | NULL | |
+----------+-------------+------+-----+---------+-------+
5 rows in set (0.01 sec)
```

5.5 更改表名

语法:  

```
ALTER TABLE 表名 RENAME 新表名
```


例如:将表emp改名为emp1

```
mysql> alter table emp rename emp1
    -> ;
Query OK, 0 rows affected (0.03 sec)
mysql> desc emp1;
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| ename    | varchar(20) | YES  |     | NULL    |       |
| age      | int(4)      | YES  |     | NULL    |       |
| hiredate | date        | YES  |     | NULL    |       |
| sal      | int(2)      | YES  |     | NULL    |       |
| deptno   | int(2)      | YES  |     | NULL    |       |
+----------+-------------+------+-----+---------+-------+
5 rows in set (0.00 sec)
```

