---
title: MYSQL表锁和事务控制
date: 2018-06-22 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---

## MYSQL表锁和事务控制



### 一.锁表

锁表语句:LOCK table 

解锁语句:UNLOCK table 



用法:

```
lock table 表名 {read [local] | [low_prioriy] WRITE}
```

<!--more-->

例如: lock table emp read; 锁定emp表

  ```
mysql> lock table emp read;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| tony  | NULL       | 1000 |   NULL |
| jerry | 2000-02-02 | 4000 |      2 |
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
| jesse | 2000-01-01 | 2000 |      1 |
+-------+------------+------+--------+
5 rows in set (0.00 sec)

  ```



开启另外一个终端.执行select查询该表.仍然可以正常查询 

```
mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| tony  | NULL       | 1000 |   NULL |
| jerry | 2000-02-02 | 4000 |      2 |
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
| jesse | 2000-01-01 | 2000 |      1 |
+-------+------------+------+--------+
5 rows in set (0.00 sec)

```



尝试修改表数据.语句被卡主,会等待表锁解除.

```
mysql> update emp set sal=500 where ename='tony';

```



解除表锁:

```
mysql> unlock tables;
Query OK, 0 rows affected (0.02 sec)
```



另外一个终端数据已经成功修改...显示语句执行了1分钟,其实就是等待锁释放的时间

```
mysql> update emp set sal=500 where ename='tony';
Query OK, 1 row affected (1 min 8.77 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```



------



### 二.事务控制

MYSQL默认情况下是自动提交的,如果需要明确的commit和rollback来提交和回滚事务.那么需要通过明确的事务控制命令来开始事务.



**事务控制语法如下:**

START TRANSACTION 或者 begin 语句开始一项新事务

COMMIT和ROLLBACK用来提交或者回滚事务

CHAIN和RELEASE字句分别用来定义在事务提交或者回滚之后的操作.CHAIN会立即启动一个新事物,release会断开和客户端连接

SET AUTOCOMMIT={0|1} 可以修改当前连接的提交方式,如果set autocommit=0 则之后的所有事务都需要通过明确命令的进行提交或者回滚



**一.演示如何使用事务控制:**



1.在终端1和终端2上查询tony的sal:

```
mysql> select sal from emp where ename='tony';

+------+
| sal  |
+------+
|  500 |
+------+
1 row in set (0.00 sec)
```



2.在终端1上用start transaction启动一个事务.把tony的sal改成1000.没有提交

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> update emp set sal=1000 where ename='tony';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 
```



3.在终端1和终端2上再次查询tony的sal;

```
mysql> select sal from emp where ename='tony';
+------+
| sal  |
+------+
| 1000 |
+------+
1 row in set (0.00 sec)

mysql> select sal from emp where ename='tony';
+------+
| sal  |
+------+
|  500 |
+------+
1 row in set (0.00 sec)
```

可以看到终端1上生效了,但是终端2上并没有生效



4.在终端1上执行提交

```
mysql> commit;

Query OK, 0 rows affected (0.03 sec)
```



5.在终端2上再次查询,发现已经成功更新

```
mysql> select sal from emp where ename='tony';
+------+
| sal  |
+------+
| 1000 |
+------+
1 row in set (0.00 sec)
```

---



**二,演示用commit and chain的效果:**



1.在终端1上开启一个事务.插入一条新数据

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into emp(ename,hiredate,sal,deptno) values('aron','2003-06-07','1500',4);
Query OK, 1 row affected (0.02 sec)
```



2.用commit and chain提交.然后再次插入一个新记录

```
mysql> commit and chain;
Query OK, 0 rows affected (0.01 sec)
mysql> insert into emp values('jenny','2004-01-01','800',4);
Query OK, 1 row affected (0.03 sec)
```



3.在终端2上查看emp表.发现刚才插入的jenny数据没有看到.

```
mysql> select * from emp where ename='aron' or ename='jenny';
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| aron  | 2003-06-07 | 1500 |      4 |
+-------+------------+------+--------+
1 row in set (0.00 sec)
```



4.在终端1上commit.然后终端2再次查看.这次看到了jenny

```
mysql> select * from emp where ename='aron' or ename='jenny';
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| aron  | 2003-06-07 | 1500 |      4 |
| jenny | 2004-01-01 |  800 |      4 |
+-------+------------+------+--------+
2 rows in set (0.00 sec)
```



------



**3.演示回滚操作**

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> update emp set sal=1000 where ename='tony';
Query OK, 1 row affected (0.01 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> select sal from emp where ename='tony';
+------+
| sal  |
+------+
| 1000 |
+------+
1 row in set (0.00 sec)

mysql> rollback;
Query OK, 0 rows affected (0.00 sec)

mysql> select sal from emp where ename='tony';
+------+
| sal  |
+------+
|  600 |
+------+
1 row in set (0.00 sec)
mysql>
```



>  note:DDL语句不能回滚.

---



**4.演示一个新事务会隐含一个Unlock table的操作**



1.对emp进行加锁

```
mysql> lock table emp write;
Query OK, 0 rows affected (0.00 sec)
```



2.终端2上查询emp表数据.select读操作被阻塞.等待锁被释放

```
mysql> select * from emp;
```



3.在终端1上开启一个start transaction新事务

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)
```



4.终端2上发现锁被释放了.可以查询了.

```
mysql> select * from emp;
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| tony  | NULL       |  600 |   NULL |
| jerry | 2000-02-02 | 4000 |      2 |
| david | 2001-02-02 | 3000 |      3 |
| jimmy | 1999-01-01 | 9000 |      3 |
| jesse | 2000-01-01 | 2000 |      1 |
| magge | 2000-02-03 | NULL |   NULL |
| aron  | 2003-06-07 | 1500 |      4 |
| jenny | 2004-01-01 |  800 |      4 |
+-------+------------+------+--------+
8 rows in set (39.31 sec)
```



------



**演示回滚部分操作**



通过定义savepoint,可以指定回滚到哪个部分(有点类似于快照).如果定义相同的savepoint名字,则后面的会覆盖前面的

release savepoint命令 可以删除一个savepoint.



1.启动一个事务,插入一条新记录

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)
mysql> insert into emp values('hebbt','2006-08-08',1300,5);
Query OK, 1 row affected (0.02 sec)
```



2.定义一个savepoint.名称为test,并且插入第二条数据,

```
mysql> savepoint test;
Query OK, 0 rows affected (0.01 sec)
mysql> insert into emp values('micheal','2006-08-08',1600,5);
Query OK, 1 row affected (0.00 sec)
```



3.查询是否有这两条记录

```
mysql> select * from emp where ename='hebbt' or ename='micheal';
+---------+------------+------+--------+
| ename   | hiredate   | sal  | deptno |
+---------+------------+------+--------+
| hebbt   | 2006-08-08 | 1300 |      5 |
| micheal | 2006-08-08 | 1600 |      5 |
+---------+------------+------+--------+
2 rows in set (0.00 sec)
```



4.在终端2上查询是否有这两条记录

```
mysql> select * from emp where ename='hebbt' or ename='micheal';
Empty set (0.00 sec)
```



5.回滚到刚才定义的savepoint.并且在终端1上再次查询.发现只有1条记录

```
mysql> rollback to savepoint test;
Query OK, 0 rows affected (0.00 sec)
mysql> select * from emp where ename='hebbt' or ename='micheal';
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| hebbt | 2006-08-08 | 1300 |      5 |
+-------+------------+------+--------+
1 row in set (0.00 sec)
```



6.commit提交,再次在终端1和终端2上查看

```
mysql> commit;
Query OK, 0 rows affected (0.01 sec)
mysql> select * from emp where ename='hebbt' or ename='micheal';
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| hebbt | 2006-08-08 | 1300 |      5 |
+-------+------------+------+--------+
1 row in set (0.00 sec)
```



在终端2上只看到commit后的第一条记录

```

mysql> select * from emp where ename='hebbt' or ename='micheal';
+-------+------------+------+--------+
| ename | hiredate   | sal  | deptno |
+-------+------------+------+--------+
| hebbt | 2006-08-08 | 1300 |      5 |
+-------+------------+------+--------+
1 row in set (0.00 sec)
```



>  savepoint之后的记录没有被commit.有点类似于快照..快照点之后的数据没有被恢复