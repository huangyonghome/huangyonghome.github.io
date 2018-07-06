---
title: MYSQL优化
date: 2018-06-22 22:59:58
tags:  mysql
categories: [mysql,mysql-basic ]
comments: true
copyright: true
---



## MYSQL优化

以下的实验都是使用mysql的案例库:sakila.这是mysql官方提供的模拟电影出租厅信息管理系统数据库.类似oracle的scott库.可以去网上下载. 

---



### 优化步骤

step1:通过show status命令了解各种SQL的执行频率 

语法:

```
show [session|global] status .如果不指定,默认为session
```

例如下面的命令显示了当前session中所有统计参数的值: 

<!--more-->

```
mysql> show session status like 'Com_%';
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| Com_admin_commands        | 0     |
| Com_assign_to_keycache    | 0     |
| Com_alter_db              | 0     |
| Com_alter_db_upgrade      | 0     |
| Com_alter_event           | 0     |
| Com_alter_function        | 0     |
| Com_alter_procedure       | 0     |
| Com_alter_server          | 0     |
| Com_alter_table           | 0     |
| Com_alter_tablespace      | 0     |
| Com_alter_user            | 0     |
| Com_analyze               | 0     |
| Com_begin                 | 0     |

```

统计信息很长,没截图完整. 以下是对部分命令的解释

Com_xxx 表示每个xxx语句执行的次数.通常比较重要的参数有:

Com_select: 执行select操作的次数,一次查询加1

Com_insert:执行insert操作的次数,对于批量插入的Insert操作,只累计一次

Com_update:

Com_delete:

> 通过上面这些不同类型的DML语句,可以看到数据库主要的操作类型是什么 



此外,show status还有其他参数可以帮助了解数据库 例如: 

- Connections:试图连接MYSQL服务器的次数 :

```
mysql> show status like 'Connections';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Connections   | 9     |
+---------------+-------+
1 row in set (0.00 sec)
```



- Slow_queries:慢查询次数 

```

mysql> show status like 'slow_que%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Slow_queries  | 0     |
+---------------+-------+
1 row in set (0.00 sec)
```



- Uptime:服务器工作时间 

```
mysql> show status like 'Uptime';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| Uptime        | 57912 |
+---------------+-------+
1 row in set (0.00 sec)
```

> 更多相关信息,请参考 show status命令.在数据库服务器操作系统上也可以使用mysqladmin extended-status 命令来获得这些信息 



---

####  step2:定位低效的SQL语句



有2种方式定位低效SQL语句.

- 通过慢查询日志定位那些执行效率较低的SQL语句.在配置文件定义long_query_time.这个参数表示大于几秒的语句被认为是执行效率低. 
- --log-slow-queries 这个参数的值为一个日志文件的路径,表示将低效SQL语句写入哪个慢日志文件.

​     

可以将以上2个配置文件,写入my.conf的配置文件中.例如:

```
[mysqld]

log="/var/lib/mysql/slow.log" 
slow_query_log = ON
log_slow_queries="/var/lib/mysql/mysql_slow.log" 
long_query_time=2
```



> 慢查询日志在查询结束后才记,默认的long_query_time时间为10S,



**使用show processlist命令可以查看当前mysql在进行的线程,线程状态,锁表等**

```
mysql> show processlist;
+----+------+-----------+--------+---------+------+-------+------------------+
| Id | User | Host      | db     | Command | Time | State | Info             |
+----+------+-----------+--------+---------+------+-------+------------------+
|  4 | root | localhost | mytest | Sleep   | 1140 |       | NULL             |
|  8 | root | localhost | NULL   | Query   |    0 | init  | show processlist |
+----+------+-----------+--------+---------+------+-------+------------------+
2 rows in set (0.00 sec)
```



SHOW PROCESSLIST显示哪些线程正在运行。您也可以使用mysqladmin processlist语句得到此信息。



**各列的含义和用途：**



**ID列**

一个标识，你要kill一个语句的时候很有用，用命令杀掉此查询 /*/mysqladmin kill 进程号。

**user列**

显示单前用户，如果不是root，这个命令就只显示你权限范围内的sql语句。

**host列**

显示这个语句是从哪个ip的哪个端口上发出的。用于追踪出问题语句的用户。

**db列**

显示这个进程目前连接的是哪个数据库。

**command列**

显示当前连接的执行的命令，一般就是休眠（sleep），查询（query），连接（connect）。

**time列**

此这个状态持续的时间，单位是秒。

**state列**

显示使用当前连接的sql语句的状态，很重要的列，后续会有所有的状态的描述，请注意，state只是语句执行中的某一个状态，一个 sql语句，以查询为例，可能需要经过copying to tmp table，Sorting result，Sending data等状态才可以完成

**info列**

显示这个sql语句，因为长度有限，所以长的sql语句就显示不全，但是一个判断问题语句的重要依据。



##### 这个命令中最关键的就是state列，mysql列出的状态主要有以下几种：

- Checking table

　正在检查数据表（这是自动的）。

- Closing tables

　正在将表中修改的数据刷新到磁盘中，同时正在关闭已经用完的表。这是一个很快的操作，如果不是这样的话，就应该确认磁盘空间是否已经满了或者磁盘是否正处于重负中。

- Connect Out

　复制从服务器正在连接主服务器。

- Copying to tmp table on disk

　由于临时结果集大于tmp_table_size，正在将临时表从内存存储转为磁盘存储以此节省内存。

- Creating tmp table

　正在创建临时表以存放部分查询结果。

- deleting from main table

　服务器正在执行多表删除中的第一部分，刚删除第一个表。

- deleting from reference tables

　服务器正在执行多表删除中的第二部分，正在删除其他表的记录。

- Flushing tables

　正在执行FLUSH TABLES，等待其他线程关闭数据表。

- Killed

　发送了一个kill请求给某线程，那么这个线程将会检查kill标志位，同时会放弃下一个kill请求。MySQL会在每次的主循环中检查kill标志位，不过有些情况下该线程可能会过一小段才能死掉。如果该线程程被其他线程锁住了，那么kill请求会在锁释放时马上生效。

- Locked

　被其他查询锁住了。

- Sending data

　正在处理SELECT查询的记录，同时正在把结果发送给客户端。

- Sorting for group

　正在为GROUP BY做排序。

　-Sorting for order

　正在为ORDER BY做排序。

- Opening tables

　这个过程应该会很快，除非受到其他因素的干扰。例如，在执ALTER TABLE或LOCK TABLE语句行完以前，数据表无法被其他线程打开。正尝试打开一个表。

- Removing duplicates

　正在执行一个SELECT DISTINCT方式的查询，但是MySQL无法在前一个阶段优化掉那些重复的记录。因此，MySQL需要再次去掉重复的记录，然后再把结果发送给客户端。

- Reopen table

　获得了对一个表的锁，但是必须在表结构修改之后才能获得这个锁。已经释放锁，关闭数据表，正尝试重新打开数据表。

- Repair by sorting

　修复指令正在排序以创建索引。

- Repair with keycache

　修复指令正在利用索引缓存一个一个地创建新索引。它会比Repair by sorting慢些。

- Searching rows for update

　正在讲符合条件的记录找出来以备更新。它必须在UPDATE要修改相关的记录之前就完成了。

- Sleeping

　正在等待客户端发送新请求.

- System lock

　正在等待取得一个外部的系统锁。如果当前没有运行多个mysqld服务器同时请求同一个表，那么可以通过增加--skip-external-locking参数来禁止外部系统锁。

- Upgrading lock

　INSERT DELAYED正在尝试取得一个锁表以插入新记录。

- Updating

　正在搜索匹配的记录，并且修改它们。

- User Lock

　正在等待GET_LOCK()。

- Waiting for tables

　该线程得到通知，数据表结构已经被修改了，需要重新打开数据表以取得新的结构。然后，为了能的重新打开数据表，必须等到所有其他线程关闭这个表。以下几种情况下会产生这个通知：

- FLUSH TABLES tbl_name,
-  ALTER TABLE, RENAME TABLE, REPAIR TABLE, ANALYZE TABLE,
- OPTIMIZE TABLE。
- waiting for handler insert

　INSERT DELAYED已经处理完了所有待处理的插入操作，正在等待新的请求。



>  大部分状态对应很快的操作，只要有一个线程保持同一个状态好几秒钟，那么可能是有问题发生了，需要检查一下。还有其他的状态没在上面中列出来，不过它们大部分只是在查看服务器是否有存在错误是才用得着。

---



#### step3 找到低效sql语句后.通过EXPLAIN命令分析SQL语句执行状态

用法:

直接在select语句钱加explain命令即可. 

例如:

```
mysql>explain select sum(amount) from customer a,payment b where 1=1 and a.customer_id=b.customer_id and email='Jane.BENNETT@sakilacustomer.org'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: a
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 599
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: b
         type: ref
possible_keys: idx_fk_customer_id
          key: idx_fk_customer_id
      key_len: 2
          ref: sakila.a.customer_id
         rows: 13
        Extra: NULL
2 rows in set (0.00 sec)
```

语法解析:

**1.select_type**: 表示select的类型.常见的有

- SIMPLE(简单表,不使用表连接或者子查询).
- PRIMARY(主查询,即外层的查询)
- UNION
- SUBQUERY(子查询的第一个select)

**2.table:**输出结果集的表

3.type:表示mysql在表中找到所需行的方式,或者叫访问类型,性能由最差到最好依次排列如下:



- ALL----全表扫描

```
mysql> explain select * from film where rating>9\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: film
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1000
        Extra: Using where
1 row in set (0.00 sec)
```

- index-----索引全扫描.MYSQL遍历整个索引查询匹配的行

```
mysql> explain select title from film\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: film
         type: index
possible_keys: NULL
          key: idx_title
      key_len: 767
          ref: NULL
         rows: 1000
        Extra: Using index
1 row in set (0.00 sec)
```

- range------索引范围扫描..常见于<,<=,>,>=等运算符操作

```
mysql> explain select * from payment where customer_id>=300 and customer_id<=350\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: payment
         type: range
possible_keys: idx_fk_customer_id
          key: idx_fk_customer_id
      key_len: 2
          ref: NULL
         rows: 1349
        Extra: Using index condition
1 row in set (0.06 sec)
```

- ref-----使用非唯一索引扫描或者唯一索引的前缀扫描.

```
mysql> explain select * from payment where customer_id=350\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: payment
         type: ref
possible_keys: idx_fk_customer_id
          key: idx_fk_customer_id
      key_len: 2
          ref: const
         rows: 23
        Extra: NULL
1 row in set (0.00 sec)
```

- idx_fk_customer_id-------非唯一索引,见上面的例子

-  eq_ref-----类似于ref.区别在于使用的索引是唯一索引.对于每个索引键值,表中只有一条记录匹配.简单来说,就是多表连接中使用primary key或者unique index作为关联条件

```
mysql> explain select * from film a,film_text b where a.film_id=b.film_id\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: a
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 1000
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: b
         type: eq_ref
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 2
          ref: sakila.a.film_id
         rows: 1
        Extra: Using index condition
2 rows in set (0.00 sec)
```

- const/system------单表中最多只有一个匹配行,查询非常迅速.例如根据主键primary key 或者 唯一索引 unique index进行查询

```
mysql> explain select * from (select * from customer where email='AARON.SELBY@sakilacustomer.org')a\G
*************************** 1. row ***************************
           id: 1
  select_type: PRIMARY
        table: <derived2>
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 599
        Extra: NULL
*************************** 2. row ***************************
           id: 2
  select_type: DERIVED
        table: customer
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 599
        Extra: Using where
2 rows in set (0.00 sec)
```

- NULL----不用访问表或者索引,直接就能得到结果



**4.possible_keys:**表示查询时可能使用的索引

**5.key:**表示实际使用的索引

**6.key_len**:使用到索引字段的长度

**7.rows:**扫描行的数量

**8.extra:**执行情况的说明和描述

通过explain extended 加上 show warnings可以看到SQL被执行前做了哪些SQL改写: 

```
mysql> explain extended  select sum(amount) from customer a,payment b where 1=1 and a.customer_id=b.customer_id and email='Jane.BENNETT@sakilacustomer.org'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: a
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 599
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: b
         type: ref
possible_keys: idx_fk_customer_id
          key: idx_fk_customer_id
      key_len: 2
          ref: sakila.a.customer_id
         rows: 13
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.00 sec)


```

