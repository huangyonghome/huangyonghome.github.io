---
title: DML操作命令
date: 2018-06-22 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---



## DML操作命令

### 简介

DML(Data Manipulation Language)-----数据库操纵语句.主要用于增删改查数据库.常用的有insert,delete,update,select

本章主要介绍mysql的DML的增删改查语句用法.

---

<!--more-->

#### 一.增(insert into)

语法:

```
INSERT INTO 表名(列名1.列名2.....) VALUES(值1,值2.....)
```

例1,向emp表插入ename为jesse,hiredate为2000-01-01,sal为2000,deptno为1:

```
mysql> insert into emp(ename,hiredate,sal,deptno)values('jesse','2000-01-01','2000',1);
Query OK, 1 row affected (0.00 sec)
```
>note:  
1.可以省略列名,直接指定values,但是值必须和表中的列名字段对应  
2.有默认值,允许为空,或者自增的列名,可以不给值

例2:只插入ename为tony,sal为1000的字段:

```
mysql> insert into emp(ename,sal)values('tony','1000');
Query OK, 1 row affected (0.00 sec)

mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jesse | 2000-01-01 | 2000 |      1 |
| tony  | NULL       | 1000 |   NULL |
+-------+------------+------+--------+
2 rows in set (0.00 sec)

```

例3.一次插入多个值

语法:

```
INSERT INTO 表名 (列名1,列名2,列名3......)values(值1,值2....),(值1,值2,.....)
```

```
mysql> create table dept(deptno int(2),deptname varchar(20));
Query OK, 0 rows affected (0.02 sec)
mysql> desc dept;
+----------+-------------+------+-----+---------+-------+
| Field    | Type        | Null | Key | Default | Extra |
+----------+-------------+------+-----+---------+-------+
| deptno   | int(2)      | YES  |     | NULL    |       |
| deptname | varchar(20) | YES  |     | NULL    |       |
+----------+-------------+------+-----+---------+-------+
2 rows in set (0.00 sec)

mysql> insert into dept values(1,'IT'),(2,'Finance'),(3,'admin'),(4,'design');
Query OK, 4 rows affected (0.01 sec)
Records: 4  Duplicates: 0  Warnings: 0

mysql> select * from dept;
+--------+----------+
| deptno | deptname |
+--------+----------+
|      1 | IT       |
|      2 | Finance  |
|      3 | admin    |
|      4 | design   |
+--------+----------+
4 rows in set (0.00 sec)

mysql> insert into emp(ename,hiredate,sal,deptno)values('jerry','2000-02-02',2000,2),('david','2001-02-02',1000,3),('jimmy','1999-01-01',3000,3);

mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jesse | 2000-01-01 | 2000 |      1 |
| tony  | NULL       | 1000 |   NULL |
| jerry | 2000-02-02 | 2000 |      2 |
| david | 2001-02-02 | 1000 |      3 |
| jimmy | 1999-01-01 | 3000 |      3 |
+-------+------------+------+--------+
5 rows in set (0.00 sec)
```
----

#### 二.改(Update 表 set)

语法:

```
UPDATE 表名 SET 列名1=值,列名2=值.......[WHERE 条件]
```

例1.将jesse的薪水从2000改成4000

```
mysql> update emp set sal='4000' where ename='jesse';
Query OK, 1 row affected (0.00 sec)

mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jesse | 2000-01-01 | 4000 |      1 |
| tony  | NULL       | 1000 |   NULL |
| jerry | 2000-02-02 | 2000 |      2 |
| david | 2001-02-02 | 1000 |      3 |
| jimmy | 1999-01-01 | 3000 |      3 |
+-------+------------+------+--------+
5 rows in set (0.00 sec)

```

同时更新多表

例2.将emp表中的每个人的sal乘以他在dept表中对应的部门ID号,另外将dept表中的employee字段改成emp表中的ename.

```
mysql> update emp a,dept b set a.sal=a.sal*b.deptno,b.employee=a.ename where a.deptno=b.deptno;
Query OK, 6 rows affected (0.01 sec)
Rows matched: 7  Changed: 6  Warnings: 0

mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jesse | 2000-01-01 | 4000 |      1 |
| tony  | NULL       | 1000 |   NULL |
| jerry | 2000-02-02 | 4000 |      2 |
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
+-------+------------+------+--------+
5 rows in set (0.00 sec)

mysql> select * from dept;
+--------+----------+----------+
| deptno | deptname | employee |
+--------+----------+----------+
|      1 | IT       | jesse    |
|      2 | Finance  | jerry    |
|      3 | admin    | david    |
|      4 | design   | NULL     |
+--------+----------+----------+
4 rows in set (0.00 sec)

```
>解析:  
将表emp设置别名为a.dept表设置别名为b.设置别名只是为了后续操作可以省略打上长长的表名,而使用别名替代a.sal 实际上就等于emp.sal 

---

#### 三.删(Delete from表)

语法:

```
DELETE FROM 表名 [where 条件]
```

>note:**如果不指定where条件,则删除表内所有记录**.....小心操作

例1:删除tony表记录

```
delete from emp where ename='tony';
```

例2.删除多个表中的多个记录

```
DELETE 1,2,3 ...... from 表1,表2,表3..... where 条件
```

例3.删除emp表中,和dept表中,deptno为1的记录

```
mysql> delete a,b from emp a,dept b where a.deptno=b.deptno and a.deptno=1;
Query OK, 2 rows affected (0.00 sec)

mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| tony  | NULL       | 1000 |   NULL |
| jerry | 2000-02-02 | 4000 |      2 |
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
+-------+------------+------+--------+
4 rows in set (0.00 sec)

mysql> select * from dept;
+--------+----------+----------+
| deptno | deptname | employee |
+--------+----------+----------+
|      2 | Finance  | jerry    |
|      3 | admin    | david    |
|      4 | design   | NULL     |
+--------+----------+----------+
3 rows in set (0.00 sec)


```
---

#### 四.查询(select * FROM 表)

语法:

```
SELECT * FROM 表名 [where 条件]
```

**1.查询不重复的记录.**

语法:

```
distinct关键字
```
例1.查询emp表中不重复的deptno部门ID

```
mysql> select distinct deptno from emp;
+--------+
| deptno |
+--------+
|   NULL |
|      2 |
|      3 |
|      1 |
+--------+
4 rows in set (0.00 sec)
```

**2.where条件查询**

这里涉及到条件判断的运算符,大概有下列运算符:

```
= 等于,<,> >=,<=,!=.or,and等
```

例2.多条件的where判断

```
mysql> select * from emp where deptno>2 and sal>2000;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
+-------+------------+------+--------+
2 rows in set (0.00 sec)
```

**3.排序和限制(关键字 ORDER BY 和 LIMIT)**

**排序**

语法:

```
SELECT * FROM 表名 [where 条件 [ORDER BY 列名1 [DESC|ASC],列名2 [DESC|ASC]]]
```

>Note:  
ORDER BY 后面跟多个列名.先按列名1进行排序,如果第一个列的值一样,也就是列名1有多个重复的值,那么这几个重复的值就按列名2的值来排序......依次类推  
DESC表示降序,ASC表示升序.默认是ASC

例3.将emp表按工资从高到低进行排序:

```
mysql> select * from emp order by sal;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| tony  | NULL       | 1000 |   NULL |
| jesse | 2000-01-01 | 2000 |      1 |
| david | 2001-02-02 | 3000 |      3 |
| jerry | 2000-02-02 | 4000 |      2 |
| jimmy | 1999-01-01 | 9000 |      3 |
+-------+------------+------+--------+
5 rows in set (0.00 sec)
```

例3-2:按deptno排序,对于重复的deptno,按工资从高到低排序

```
mysql> select * from emp order by deptno,sal desc;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| tony  | NULL       | 1000 |   NULL |
| jesse | 2000-01-01 | 2000 |      1 |
| jerry | 2000-02-02 | 4000 |      2 |
| jimmy | 1999-01-01 | 9000 |      3 |
| david | 2001-02-02 | 3000 |      3 |
+-------+------------+------+--------+
5 rows in set (0.00 sec)
```

**Limit**

如果对于排序后的输出,只显示一部分,而不是全部.就可以使用Limit关键字,表示限制输出的行数

语法:

```
limit 初始偏移量(默认为0),限制行数
```

例3-3:对工资进行排序,并且只显示3行

```
mysql> select * from emp order by sal limit 3;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| tony  | NULL       | 1000 |   NULL |
| jesse | 2000-01-01 | 2000 |      1 |
| david | 2001-02-02 | 3000 |      3 |
+-------+------------+------+--------+
3 rows in set (0.00 sec)
```

例3-4:从第二行开始显示,显示3行..(偏移量设置为1)

```
mysql> select * from emp order by sal limit 1,3;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jesse | 2000-01-01 | 2000 |      1 |
| david | 2001-02-02 | 3000 |      3 |
| jerry | 2000-02-02 | 4000 |      2 |
+-------+------------+------+--------+
3 rows in set (0.00 sec)
```

**4.聚合 (GROUP BY, WITH, HAVING关键字)**

聚合用来做汇总操作.语法如下:

```
SELECT [列名1,列名2] 聚合操作 FROM 表名 [WHERE 条件] [GROUP BY 列名1,列名2.....] [with ROLLUP] [HAVING  条件]
```

聚合操作也就是一些聚合函数.常用的有 sum求和,count(*)记录数,MAX,MIN  
GROUP BY 表示要进行聚合的字段,如果按照部门分类,那么就是group by 部门  
WITH ROLLUP 表示是否对分类聚合后的结果再汇总  
HAVING 表示对分类后的结果再进行条件过滤  

例4:查看emp表中的总人数

```
mysql> select count(1) from emp;
+----------+
| count(1) |
+----------+
|        5 |
+----------+
1 row in set (0.00 sec)
```

例4-1:统计deptno部门ID中各部门的人数

```
mysql> select deptno,count(1) from emp group by deptno;
+--------+----------+
| deptno | count(1) |
+--------+----------+
|   NULL |        1 |
|      1 |        1 |
|      2 |        1 |
|      3 |        2 |
+--------+----------+
4 rows in set (0.00 sec)
```

例4-2:既统计各部门人数,又统计总人数

```
mysql> select deptno,count(1) from emp group by deptno with rollup;
+--------+----------+
| deptno | count(1) |
+--------+----------+
|   NULL |        1 |
|      1 |        1 |
|      2 |        1 |
|      3 |        2 |
|   NULL |        5 |
+--------+----------+
5 rows in set (0.00 sec)

```
例4-3:统计人数大于1的部门ID

```
mysql> select deptno,count(1) from emp group by deptno having  count(1)>1;
+--------+----------+
| deptno | count(1) |
+--------+----------+
|      3 |        2 |
+--------+----------+
1 row in set (0.00 sec)
```

例4-4:统计公司所有员工的薪水总额,最高薪水,最低薪水

```
mysql> select sum(sal),max(sal),min(sal) from emp;
+----------+----------+----------+
| sum(sal) | max(sal) | min(sal) |
+----------+----------+----------+
|    19000 |     9000 |     1000 |
+----------+----------+----------+
1 row in set (0.00 sec)
```

**5.表连接 (关键字 left join,right join)**

表连接分为内连接和外链接

**内连接**:最常用,仅显示两张表匹配的记录,  
**外连接**----又分为左连接,右连接  

**左连接**:显示所有左表中的记录,以及与之匹配的右表记录.---关键字:left join  
**右连接**:显示所有的右表记录,以及与之匹配的左表记录----关键字:right


外连接格式:

```
select * from 左表名 left|right join 右表名 on 左表字段=右表字段
```

内连接用法刚刚演示过了.比如查询emp表中的ename,dept表中的deptname

例5:

```
mysql> select ename,deptname from emp,dept where emp.deptno=dept.deptno;
+-------+----------+
| ename | deptname |
+-------+----------+
| jerry | Finance  |
| david | admin    |
| jimmy | admin    |
| jesse | IT       |
+-------+----------+
4 rows in set (0.00 sec)
```

左连接: 还是刚刚的例子,但是这次会显示emp表中,没有和dept表中deptno字段匹配到的其他记录

例5-1:

```
mysql> select ename,deptname from emp left join dept on  emp.deptno=dept.deptno;
+-------+----------+
| ename | deptname |
+-------+----------+
| jerry | Finance  |
| david | admin    |
| jimmy | admin    |
| jesse | IT       |
| tony  | NULL     |
+-------+----------+
5 rows in set (0.00 sec)

这里把没有匹配到的左表的tony显示出来了.右连接情况类似,不再赘述
```

**6.子查询 (关键字 in,not in, =,!=,exists,not exists)**

子查询表示当进行查询的时候,where 的条件是另外一个查询的结果.

例6.在emp表中查询在dept表中有记录的deptno.

```
mysql> select * from emp where deptno in(select deptno from dept);
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jerry | 2000-02-02 | 4000 |      2 |
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
| jesse | 2000-01-01 | 2000 |      1 |
+-------+------------+------+--------+
4 rows in set (0.00 sec)
```

如果子查询的记录数是唯一的,那么可以用=替代in.例6-1:

```
mysql> select * from emp where deptno =(select deptno from dept where deptno=1);
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jesse | 2000-01-01 | 2000 |      1 |
+-------+------------+------+--------+
1 row in set (0.00 sec)

或者:

mysql> select * from emp where deptno =(select deptno from dept limit 1);
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jerry | 2000-02-02 | 4000 |      2 |
+-------+------------+------+--------+
1 row in set (0.00 sec)
```

子查询也可以转化成表连接,刚才的例子用表连接同样可以实现.例如6-2:

```
mysql> select emp.* from emp,dept where emp.deptno=dept.deptno;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| jerry | 2000-02-02 | 4000 |      2 |
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
| jesse | 2000-01-01 | 2000 |      1 |
+-------+------------+------+--------+
4 rows in set (0.00 sec)
```

**7.记录联合 (关键字,union,union all)**

记录联合用来将多个表的数据按照一定的查询条件查询出来后,将结果合并一起输出.  
union表示输出的结果进行一个distinct去除重复数据.  
union all表示显示联合后的所有信息

语法: 

```
SELECT * FROM 表1 UNION|UNION ALL SELECT * FROM 表2 .............................一直到表N
```

例7:将emp和dept的部门编号集合显示:

```
mysql> select deptno from emp
    -> union
    -> select deptno from dept;
+--------+
| deptno |
+--------+
|   NULL |
|      2 |
|      3 |
|      1 |
|      4 |
+--------+
5 rows in set (0.00 sec)
```



