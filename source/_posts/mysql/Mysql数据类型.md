---
title: Mysql数据类型
date: 2018-06-22 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---



## Mysql数据类型



#### 一.数值类型

**1.整数类型:**

- TINYINIT  
- SMALLINT  
- mediumINT  
- INT,integer  
- bigINT  

整数后可以跟指定一个小括号(),小括号内指定显示宽度.例如Int(5).表示当数值宽度小于5位数的时候,在数字前填0充满宽度

另外还有auto_increment来表示一个某个列下的值自增,一个表中只能有一个自增列.

对于自增列,应该定义为Not null,或者primary key 或者unique键

<!--more-->

**2.浮点数类型**

- float 单精度
- double 双精度
- decimal 定点数

浮点数和定点数可以在类型后面加个小括号(M,D)表示..M表示小数的总共位数(包括整数+小数位).D表示小数位..例如float(5,3)表示2个整数,3个小数.,例如:12.345

mysql在保存小数时,自动进行四舍五入

也可以不加小括号().代表不指定精度.浮点数如果不写精度,则按实际精度值显示,定点数如果不指定精度,则按照默认值decimal(10,0)来操作.如果数据超越了精度,则报错



**3.位数**

- ​    BIT

---



#### 二.时间类型

- DATE---表示年月日(常用)
- DATETIME---表示年月日时分秒(常用).DATETIME是DATE和TIME的组合
- TIME---------表示时分秒(常用)
- YEAR-----表示年份
- TIMESTAMP---表示当前系统时间.插入一个NULL时,MYSQL会自动替换成当前系统时间,但是仅限于一个TIMESTAMP

---



#### 三.字符串

- CHAR--------字符串长度为0-255之间整数
- VARCHAR---字符串长度为0-65535整数
- BINARY-------字符串被保存为2进制
- VARBINARY---同上
- BLOB---------
- TEXT
- ENUM---------枚举
- SET