---
title: MYSQL常用函数
date: 2018-06-22 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---



## MYSQL常用函数 



#### 一.字符串函数 



**1.concat(s1,s2.....)--------连接**

s1,s2....为一个字符串

<!--more-->

例如:连接以下三个字符串

```
mysql> select concat('abc','def','hij');
+---------------------------+
| concat('abc','def','hij') |
+---------------------------+
| abcdefhij                 |
+---------------------------+
1 row in set (0.00 sec)

```



**2.insert(str,x,y,instr) **

说明:

将字符串str从第x位置开始,y个字符长的字符串替换为字符串instr 

例如.将字符串abcde从第二个位置开始替换成haha

```
mysql> select insert('abcde',2,4,'haha');
+----------------------------+
| insert('abcde',2,4,'haha') |
+----------------------------+
| ahaha                      |
+----------------------------+
1 row in set (0.00 sec)

```



**3.lower(str),upper(str) **

说明:

字符串大小写切换 



**4.left(str,x)**

说明:

返回字符串str最左边的x个字符

例如:返回abcde最左边的2个字符

```
mysql> select left ('abcde',2);
+------------------+
| left ('abcde',2) |
+------------------+
| ab               |
+------------------+
1 row in set (0.00 sec)
```



**5.right(str,x)**

说明:

返回字符串str最右边的x个字符

例如:

```
mysql> select right ('abcde',2);
+-------------------+
| right ('abcde',2) |
+-------------------+
| de                |
+-------------------+
1 row in set (0.00 sec)
```



**6.LPAD(str,n,pad)和rpad(str,n,pad)**

说明:

LPAD:用字符串pad对str最左边进行填充,直到长度为n个字符长度

rpad:用字符串pad对str最右边进行填充,直到长度为n个字符长度

> 这里的n表示填充后字符的总长度

例如:

```
mysql> select lpad('abcd',8,'haha');
+-----------------------+
| lpad('abcd',8,'haha') |
+-----------------------+
| hahaabcd              |
+-----------------------+
1 row in set (0.00 sec)

mysql> select rpad('abcd',8,'haha');
+-----------------------+
| rpad('abcd',8,'haha') |
+-----------------------+
| abcdhaha              |
+-----------------------+
1 row in set (0.00 sec)
```



**7.LTRIM(str) 和RTRIM(str) **

说明:

LTRIM:去掉字符串str左侧的空格 

RTRIM:去掉字符串str右侧的空格 



例如:

```
mysql> select ltrim('   abcd');
+------------------+
| ltrim('   abcd') |
+------------------+
| abcd             |
+------------------+
1 row in set (0.00 sec)
```



**8.REPEAT(str,x) **

说明:

返回字符串str重复x次的结果 

例如

```
mysql> select repeat('abcd',3);
+------------------+
| repeat('abcd',3) |
+------------------+
| abcdabcdabcd     |
+------------------+
1 row in set (0.00 sec)
```



**9.REPLACE(str,a,b) **

说明:

用字符串b替换字符串str中出现的字符串a 

例如:

```
mysql> select replace('abcd200','200','2018');
+---------------------------------+
| replace('abcd200','200','2018') |
+---------------------------------+
| abcd2018                        |
+---------------------------------+
1 row in set (0.00 sec)
```



**10.TRIM(str) **

说明:

去掉字符串行尾和行首的空格 

例如:

```
mysql> select trim('    abcd     ');
+-----------------------+
| trim('    abcd     ') |
+-----------------------+
| abcd                  |
+-----------------------+
1 row in set (0.00 sec)
```



**11.SUBSTRING('str',x,y) **

说明:

返回从字符串str x位置起y个字符长度的字符串 



例如:

```
mysql> select substring('abcde200',6,3);
+---------------------------+
| substring('abcde200',6,3) |
+---------------------------+
| 200                       |
+---------------------------+
1 row in set (0.00 sec)
```

---

### 数值函数



**1.ABS(x) **

说明:

返回x的绝对值 

例如:

```
mysql> select abs(2);
+--------+
| abs(2) |
+--------+
|      2 |
+--------+
1 row in set (0.00 sec)
mysql> select abs(2.4);
+----------+
| abs(2.4) |
+----------+
|      2.4 |
+----------+
1 row in set (0.00 sec)
```



**2.CEIL(x) 和FLOOR(x) **

说明:

- CEIL(x)--------返回大于x的最小整数值
- FLOOR(x)-----返回小于x的最大整数值

例如:

```
mysql> select ceil(1.4),floor(1.4);
+-----------+------------+
| ceil(1.4) | floor(1.4) |
+-----------+------------+
|         2 |          1 |
+-----------+------------+
1 row in set (0.00 sec)
```



**3.MOD(x,y) **

说明:

返回x/y的模, 



**4.RAND() **

说明:

返回0-1内的随机数 

例如:

```
mysql> select rand(),rand();
+---------------------+--------------------+
| rand()              | rand()             |
+---------------------+--------------------+
| 0.14972682963099967 | 0.5293732364935552 |
+---------------------+--------------------+
1 row in set (0.01 sec)
```

> note:如果需要获得一个100以内的随机数,可以进行如下操作: 
>
> ```
> mysql> select ceil(100*rand()),ceil(100*rand());
> +------------------+------------------+
> | ceil(100*rand()) | ceil(100*rand()) |
> +------------------+------------------+
> |               76 |               85 |
> +------------------+------------------+
> 1 row in set (0.00 sec)
> ```



**5.round(x,y) **

用法:

返回x的四舍五入有y位小数的值 

例如:

```
mysql> select round(3.12345,3), round(3.1456,2);
+--------------------+-------------------+
| round(3.12345,'3') | round(3.1456,'2') |
+--------------------+-------------------+
|              3.123 |              3.15 |
+--------------------+-------------------+
1 row in set (0.00 sec)
```



**6.truncate(x,y) **

用法:

返回数字x截断为y位小数的结果 

例如:

```
mysql> select truncate(3.1234,3);
+--------------------+
| truncate(3.1234,3) |
+--------------------+
|              3.123 |
+--------------------+
1 row in set (0.00 sec)
```

> note:truncate值是单纯的截断小数位.而round截断的时候会自动四舍五入. 例如:
>
> ```
> mysql> select round(3.1456,'2'),truncate(3.1456,2);
> +-------------------+--------------------+
> | round(3.1456,'2') | truncate(3.1456,2) |
> +-------------------+--------------------+
> |              3.15 |               3.14 |
> +-------------------+--------------------+
> 1 row in set (0.00 sec)
> ```



---

### 三.日期函数

- CURDATE()--------返回当前日期

```
mysql> select curdate();
+------------+
| curdate()  |
+------------+
| 2017-12-31 |
+------------+
1 row in set (0.00 sec)
```



- CURTIME()---------返回当前时间

```
mysql> select curtime();
+-----------+
| curtime() |
+-----------+
| 10:38:31  |
+-----------+
1 row in set (0.00 sec)
```



- NOW()--------------返回当前的日期和时间

```
mysql> select now();
+---------------------+
| now()               |
+---------------------+
| 2017-12-31 10:40:57 |
+---------------------+
1 row in set (0.00 sec)
```



- UNIX_timestamp(date)---返回日期date的UNIX时间戳

- from_unixtime------返回unix时间戳的日期

  

- week(date)-----返回date为一年中的第几个星期

- year(date)-------返回日期date的年份

```
mysql> select week(now()),year(now());
+-------------+-------------+
| week(now()) | year(now()) |
+-------------+-------------+
|          53 |        2017 |
+-------------+-------------+
1 row in set (0.00 sec)
```



- HOUR(time)------返回time的小时值
- minute(time)--------返回time的分钟值

```
mysql> select hour(now()),minute(now());
+-------------+---------------+
| hour(now()) | minute(now()) |
+-------------+---------------+
|          10 |            47 |
+-------------+---------------+
1 row in set (0.00 sec)
```



- monthname(date)-----返回date的月份名

```
mysql> select monthname(now());
+------------------+
| monthname(now()) |
+------------------+
| December         |
+------------------+
1 row in set (0.00 sec)
```



- date_format(date,fmt)--------返回按字符串fmt格式化日期date值
- date_add(date,INTERVAL expr type)--------返回与所给日期date相差interval时间的日期 

```
mysql> select now() current,date_add(now(),interval 31 day) after31days,date_add(now(),interval -31 day) before31days;
+---------------------+---------------------+---------------------+
| current             | after31days         | before31days        |
+---------------------+---------------------+---------------------+
| 2017-12-31 10:54:18 | 2018-01-31 10:54:18 | 2017-11-30 10:54:18 |
+---------------------+---------------------+---------------------+
1 row in set (0.00 sec)
```



- datediff(expr,expr2)---------返回起始时间expr和结束时间expr2之间的天数

```
mysql> select datediff('2028-12-31',now());
+------------------------------+
| datediff('2028-12-31',now()) |
+------------------------------+
|                         4018 |
+------------------------------+
1 row in set (0.00 sec)
```

---



### 四.流程函数 



- if(value,t,f)---------如果value是真,返回t,否则返回f



例如:如果工资超过2000的,就用'high'表示,否则用'low'---- 

```
mysql> select if(sal>2000,'high','low') from emp;
+---------------------------+
| if(sal>2000,'high','low') |
+---------------------------+
| low                       |
| high                      |
| high                      |
| high                      |
| low                       |
+---------------------------+
5 rows in set (0.01 sec)
```



- ifnull(value1,value2)--------用来替换Null值,如果值为Null则显示为0

```
mysql> select ifnull(sal,0) from emp;
+---------------+
| ifnull(sal,0) |
+---------------+
|          1000 |
|          4000 |
|          3000 |
|          9000 |
|          2000 |
|             0 |
+---------------+
6 rows in set (0.00 sec)
```



- case when [value1] then [result1]  when[value2] then [result2]..............else[default]END



例如将刚才工资分成高低档 :

```
mysql> select case when sal>=2000 then 'high' else 'low' end from emp;
+------------------------------------------------+
| case when sal>=2000 then 'high' else 'low' end |
+------------------------------------------------+
| low                                            |
| high                                           |
| high                                           |
| high                                           |
| high                                           |
| low                                            |
+------------------------------------------------+
6 rows in set (0.00 sec)
```

或者分3档:

```
mysql> select case when sal<2000 then 'low' when sal>=2000 and sal<=4000 then 'medium' when sal>=4000 then 'high' end from emp;
+---------------------------------------------------------------------------------------------------------+
| case when sal<2000 then 'low' when sal>=2000 and sal<=4000 then 'medium' when sal>=4000 then 'high' end |
+---------------------------------------------------------------------------------------------------------+
| low                                                                                                     |
| medium                                                                                                  |
| medium                                                                                                  |
| high                                                                                                    |
| medium                                                                                                  |
| NULL                                                                                                    |
+---------------------------------------------------------------------------------------------------------+
6 rows in set (0.00 sec)
```



**有其他一些常用函数.比如:**



- database()------查看当前数据库名
- version()---------查看数据库版本
- user()------------返回当前登录用户名
- inet_aton(IP)----返回IP地址的数字表示
- inet_ntoa(num)--返回数字代表的IP地址
- password(str)----返回字符串的加密版本
- md5()-----返回字符串的md5值

