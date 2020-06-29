---
title: mysql5.7主从同步延时问题
date: 2018-08-22 11:59:58
tags:  [mysql,mysql维护 ]
categories: [mysql,mysql维护 ]
comments: true
copyright: true
---

## mysql主从同步延时问题

最近领导将MASTER的主库清空了最近几个月的数据,进行了大并发的操作.这导致了mysql的从库延时非常高的问题.zabbix报警如下:

![mysql-zabbix](https://img1.jesse.top/mysql-monitor.png)

延时一直飙升到23个小时.

<!--more--> 

#### 解决过程

1.查看master库的慢查询条数:

```
"root@localhost:mysql.sock  [(none)]>show processlist;
+-----------+--------+----------------------+---------------+-------------+--------+---------------------------------------------------------------+------------------+
| Id        | User   | Host                 | db            | Command     | Time   | State                                                         | Info             |
+-----------+--------+----------------------+---------------+-------------+--------+---------------------------------------------------------------+------------------+
                                                         | NULL             |
| 285241295 | repl   | 10.8.0.6:60866       | NULL          | Binlog Dump | 319038 | Master has sent all binlog to slave; waiting for more updates | NULL             |
| 285243495 | canal  | 10.25.2.85:42252     | NULL          | Binlog Dump | 319014 | Master has sent all binlog to slave; waiting for more updates | NULL             |
| 290884712 | root   | 10.47.54.80:49170    | dwd_cron      | Sleep       |     68 |                                                               | NULL             |
| 292617828 | tongji | 10.153.138.121:62821 | NULL          | Sleep       |      1 |                                                               | NULL             |
| 292617838 | tongji | 10.153.138.121:62822 | NULL          | Sleep       |      2 |                                                               | NULL             |
| 292617918 | tongji | 10.153.138.121:62872 | NULL          | Sleep       |      2 |                                                               | NULL             |
| 300508313 | root   | 10.47.54.80:41858    | dwd_cron      | Sleep       |    194 |                                                               | NULL             |
| 309403589 | root   | 10.47.54.80:42136    | dwd_cron      | Sleep       |    198 |                                                               | NULL             |
| 309403590 | root   | 10.47.54.80:42138    | dwd_cron      | Sleep       |    198 |                                                               | NULL             |
| 313939869 | root   | 10.25.2.85:40936     | hsq_online    | Sleep       |     11 |                                                               | NULL             |
| 317005283 | root   | 10.25.2.85:34854     | hsq_online    | Sleep       |  27800 |                                                               | NULL             |                                                               | NULL             |
| 319272252 | repl   | 10.27.3.27:35818     | NULL          | Binlog Dump |   3014 | Master has sent all binlog to slave; waiting for more updates | NULL             |
                                                             | NULL             |
+-----------+--------+----------------------+---------------+-------------+--------+---------------------------------------------------------------+------------------+
118 rows in set (0.01 sec)

```

只截取了部分数据..慢查询条数并不多,而且binlog也已经全部send到slave了..主库这边一切正常.



2.查看从库的状态

```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.81.61.101
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.005022
          Read_Master_Log_Pos: 905366832
               Relay_Log_File: server-6-relay-bin.004891
                Relay_Log_Pos: 925220397
        Relay_Master_Log_File: mysql-bin.004927
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: dwd_analystic,hsq_sync_RDS
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 925220184
              Relay_Log_Space: 102979799744
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 84227
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 63306

```

除了Seconds_Behind_Master参数的值非常高以外也没有太大问题



3.查看从库的慢查询情况

```
mysql> show processlist;
+--------+-------------+----------------+--------------+---------+--------+----------------------------------+------------------------------------------------------------------------------------------------------+
| Id     | User        | Host           | db           | Command | Time   | State                            | Info  |
+--------+-------------+----------------+--------------+---------+--------+----------------------------------+------------------------------------------------------------------------------------------------------+
| 358262 | system user |                | NULL         | Connect | 319299 | Waiting for master to send event | NULL  |
| 358263 | system user |                | NULL         | Connect |  84290 | System lock                      | NULL  |
| 433117 | tongji      | server-5:53032 | hsq_sync_RDS | Query   |   1715 | Sending data                     | SELECT `id`, `user_id`, `buyer_id`, `order_ids`, `payment_method`, `trade_no`, `tn`, `third_party_no |
| 433118 | tongji      | server-2:54130 | hsq_sync_RDS | Query   |   1714 | Sending data                     | SELECT `id`, `user_id`, `buyer_id`, `order_ids`, `payment_method`, `trade_no`, `tn`, `third_party_no |
| 433119 | tongji      | server-1:34396 | hsq_sync_RDS | Query   |   1714 | Sending data                     | SELECT `id`, `user_id`, `buyer_id`, `order_ids`, `payment_method`, `trade_no`, `tn`, `third_party_no |
| 433120 | tongji      | server-6:45128 | hsq_sync_RDS | Query   |   1712 | Sending data                     | SELECT `id`, `user_id`, `buyer_id`, `order_ids`, `payment_method`, `trade_no`, `tn`, `third_party_no |
| 433181 | tongji      | server-1:34744 | hsq_sync_RDS | Query   |   1626 | Sending data                     | SELECT `id`, `user_id`, `password`, `last_login_ip`, `token`, `session`, `mobile`, `email`, `wechat_ |
| 433206 | tongji      | server-6:45812 | hsq_sync_RDS | Query   |   1587 | Sending data                     | SELECT `id`, `order_id`, `user_id`, `delivery_no`, `delivery_com_code`, `delivery_com_name`, `status |
| 433207 | tongji      | server-2:54888 | hsq_sync_RDS | Query   |   1587 | Sending data                     | SELECT `id`, `order_id`, `user_id`, `delivery_no`, `delivery_com_code`, `delivery_com_name`, `status |
| 433208 | tongji      | server-4:47770 | hsq_sync_RDS | Query   |   1587 | Sending data                     | SELECT `id`, `order_id`, `user_id`, `delivery_no`, `delivery_com_code`, `delivery_com_name`, `status |
| 433209 | tongji      | server-1:35058 | hsq_sync_RDS | Query   |   1586 | Sending data                     | SELECT `id`, `order_id`, `user_id`, `delivery_no`, `delivery_com_code`, `delivery_com_name`, `status |
| 433246 | root        | localhost      | NULL         | Query   |      0 | starting                         | show processlist  |
| 433373 | tongji      | server-6:47244 | hsq_sync_RDS | Query   |   1281 | Sending data                     | SELECT `id`, `user_id`, `password`, `last_login_ip`, `token`, `session`, `mobile`, `email`, `wechat_ |
| 433505 | tongji      | server-1:38008 | hsq_sync_RDS | Query   |    797 | Sending data                     | SELECT `id`, `user_id`, `open_id`, `type`, `app_id`, `access_token`, `refresh_token`, `token_refresh |
| 433506 | tongji      | server-5:34952 | hsq_sync_RDS | Query   |    797 | Sending data                     | SELECT `id`, `user_id`, `open_id`, `type`, `app_id`, `access_token`, `refresh_token`, `token_refresh |
| 433508 | tongji      | server-3:33080 | hsq_sync_RDS | Query   |    796 | Sending data                     | SELECT `id`, `user_id`, `open_id`, `type`, `app_id`, `access_token`, `refresh_token`, `token_refresh |
| 433977 | tongji      | server-3:38782 | hsq_sync_RDS | Query   |     53 | Sending data                     | SELECT `id`, `user_id`, `open_id`, `type`, `app_id`, `access_token`, `refresh_token`, `token_refresh |
| 433982 | tongji      | server-3:38948 | hsq_sync_RDS | Query   |     40 | Sending data                     | SELECT `payment_id`, `order_id` FROM `trade_order_payment_map` AS `trade_order_payment_map` WHERE (  |
| 433983 | tongji      | server-2:35990 | hsq_sync_RDS | Query   |     40 | Sending data                     | SELECT `payment_id`, `order_id` FROM `trade_order_payment_map` AS `trade_order_payment_map` WHERE (  |
| 433984 | tongji      | server-6:53400 | hsq_sync_RDS | Query   |     40 | Sending data                     | SELECT `payment_id`, `order_id` FROM `trade_order_payment_map` AS `trade_order_payment_map` WHERE (  |
| 433985 | tongji      | server-4:44578 | hsq_sync_RDS | Query   |     40 | Sending data                     | SELECT `payment_id`, `order_id` FROM `trade_order_payment_map` AS `trade_order_payment_map` WHERE (  |
| 433986 | tongji      | server-3:38986 | hsq_sync_RDS | Query   |     38 | Sending data                     | SELECT `id`, `user_id`, `open_id`, `type`, `app_id`, `access_token`, `refresh_token`, `token_refresh |
| 433997 | tongji      | server-2:36126 | hsq_sync_RDS | Query   |     23 | Sending data                     | SELECT `id`, `user_id`, `open_id`, `type`, `app_id`, `access_token`, `refresh_token`, `token_refresh |
| 434019 | tongji      | server-6:53648 | hsq_sync_RDS | Sleep   |      0 |                                  | NULL  |
+--------+-------------+----------------+--------------+---------+--------+----------------------------------+------------------------------------------------------------------------------------------------------+
24 rows in set (0.00 sec)

```

从库的select语句性能不是很好,

mysql数据文件目录存在大量的binlog日志.显然从库的数据写入有严重的滞后问题

```
-rw-r----- 1 mysql mysql        257 Aug 19 11:04 server-6-relay-bin.004893
-rw-r----- 1 mysql mysql 1074907748 Aug 19 11:20 server-6-relay-bin.004894
-rw-r----- 1 mysql mysql     326897 Aug 19 11:20 server-6-relay-bin.004895
-rw-r----- 1 mysql mysql        257 Aug 19 11:20 server-6-relay-bin.004896
-rw-r----- 1 mysql mysql 1073757574 Aug 19 11:35 server-6-relay-bin.004897
-rw-r----- 1 mysql mysql    3014623 Aug 19 11:35 server-6-relay-bin.004898
-rw-r----- 1 mysql mysql        257 Aug 19 11:35 server-6-relay-bin.004899
-rw-r----- 1 mysql mysql 1073756933 Aug 19 11:50 server-6-relay-bin.004900
-rw-r----- 1 mysql mysql       1981 Aug 19 11:50 server-6-relay-bin.004901
-rw-r----- 1 mysql mysql        257 Aug 19 11:50 server-6-relay-bin.004902
-rw-r----- 1 mysql mysql 1073755064 Aug 19 12:05 server-6-relay-bin.004903
-rw-r----- 1 mysql mysql     461835 Aug 19 12:05 server-6-relay-bin.004904
-rw-r----- 1 mysql mysql        257 Aug 19 12:05 server-6-relay-bin.004905
-rw-r----- 1 mysql mysql 1074493379 Aug 19 12:20 server-6-relay-bin.004906
-rw-r----- 1 mysql mysql       2579 Aug 19 12:20 server-6-relay-bin.004907
-rw-r----- 1 mysql mysql        257 Aug 19 12:20 server-6-relay-bin.004908
-rw-r----- 1 mysql mysql 1073750403 Aug 19 12:35 server-6-relay-bin.004909
-rw-r----- 1 mysql mysql      38029 Aug 19 12:35 server-6-relay-bin.004910
-rw-r----- 1 mysql mysql        257 Aug 19 12:35 server-6-relay-bin.004911
-rw-r----- 1 mysql mysql 1073757889 Aug 19 12:50 server-6-relay-bin.004912
-rw-r----- 1 mysql mysql     472751 Aug 19 12:50 server-6-relay-bin.004913
-rw-r----- 1 mysql mysql        257 Aug 19 12:50 server-6-relay-bin.004914
-rw-r----- 1 mysql mysql 1075262040 Aug 19 13:00 server-6-relay-bin.004915
-rw-r----- 1 mysql mysql       3009 Aug 19 13:00 server-6-relay-bin.004916
-rw-r----- 1 mysql mysql        257 Aug 19 13:00 server-6-relay-bin.004917
-rw-r----- 1 mysql mysql 1073747998 Aug 19 13:15 server-6-relay-bin.004918
-rw-r----- 1 mysql mysql     421255 Aug 19 13:15 server-6-relay-bin.004919
-rw-r----- 1 mysql mysql        257 Aug 19 13:15 server-6-relay-bin.004920
-rw-r----- 1 mysql mysql 1073742443 Aug 19 13:31 server-6-relay-bin.004921
-rw-r----- 1 mysql mysql      21636 Aug 19 13:31 server-6-relay-bin.004922
-rw-r----- 1 mysql mysql        257 Aug 19 13:31 server-6-relay-bin.004923
-rw-r----- 1 mysql mysql 1074325521 Aug 19 13:47 server-6-relay-bin.004924
-rw-r----- 1 mysql mysql       8141 Aug 19 13:47 server-6-relay-bin.004925
-rw-r----- 1 mysql mysql        257 Aug 19 13:47 server-6-relay-bin.004926
-rw-r----- 1 mysql mysql 1074539167 Aug 19 14:01 server-6-relay-bin.004927
-rw-r----- 1 mysql mysql       3783 Aug 19 14:01 server-6-relay-bin.004928
-rw-r----- 1 mysql mysql        257 Aug 19 14:01 server-6-relay-bin.004929
-rw-r----- 1 mysql mysql 1073756221 Aug 19 14:15 server-6-relay-bin.004930
-rw-r----- 1 mysql mysql     505231 Aug 19 14:15 server-6-relay-bin.004931
-rw-r----- 1 mysql mysql        257 Aug 19 14:15 server-6-relay-bin.004932
-rw-r----- 1 mysql mysql 1073741891 Aug 19 14:32 server-6-relay-bin.004933
-rw-r----- 1 mysql mysql    2159339 Aug 19 14:32 server-6-relay-bin.004934
-rw-r----- 1 mysql mysql        257 Aug 19 14:32 server-6-relay-bin.004935
-rw-r----- 1 mysql mysql 1074633232 Aug 19 14:48 server-6-relay-bin.004936
-rw-r----- 1 mysql mysql       5415 Aug 19 14:48 server-6-relay-bin.004937
-rw-r----- 1 mysql mysql        257 Aug 19 14:48 server-6-relay-bin.004938
-rw-r----- 1 mysql mysql 1074746727 Aug 19 15:03 server-6-relay-bin.004939
-rw-r----- 1 mysql mysql       2791 Aug 19 15:03 server-6-relay-bin.004940
-rw-r----- 1 mysql mysql        257 Aug 19 15:03 server-6-relay-bin.004941
-rw-r----- 1 mysql mysql 1074641101 Aug 19 15:18 server-6-relay-bin.004942
-rw-r----- 1 mysql mysql        329 Aug 19 15:18 server-6-relay-bin.004943
-rw-r----- 1 mysql mysql        257 Aug 19 15:18 server-6-relay-bin.004944
-rw-r----- 1 mysql mysql 1074670108 Aug 19 15:35 server-6-relay-bin.004945
-rw-r----- 1 mysql mysql     544530 Aug 19 15:35 server-6-relay-bin.004946
-rw-r----- 1 mysql mysql        257 Aug 19 15:35 server-6-relay-bin.004947
-rw-r----- 1 mysql mysql 1075095236 Aug 19 15:50 server-6-relay-bin.004948
-rw-r----- 1 mysql mysql     370712 Aug 19 15:50 server-6-relay-bin.004949
-rw-r----- 1 mysql mysql        257 Aug 19 15:50 server-6-relay-bin.004950
-rw-r----- 1 mysql mysql 1075138343 Aug 19 16:01 server-6-relay-bin.004951
-rw-r----- 1 mysql mysql       1148 Aug 19 16:01 server-6-relay-bin.004952
-rw-r----- 1 mysql mysql        257 Aug 19 16:01 server-6-relay-bin.004953
-rw-r----- 1 mysql mysql 1073753579 Aug 19 16:15 server-6-relay-bin.004954
-rw-r----- 1 mysql mysql     166719 Aug 19 16:15 server-6-relay-bin.004955
-rw-r----- 1 mysql mysql        257 Aug 19 16:15 server-6-relay-bin.004956
-rw-r----- 1 mysql mysql 1073744509 Aug 19 16:31 server-6-relay-bin.004957
-rw-r----- 1 mysql mysql        329 Aug 19 16:31 server-6-relay-bin.004958
-rw-r----- 1 mysql mysql        257 Aug 19 16:31 server-6-relay-bin.004959
-rw-r----- 1 mysql mysql 1075785150 Aug 19 16:48 server-6-relay-bin.004960
-rw-r----- 1 mysql mysql       3213 Aug 19 16:48 server-6-relay-bin.004961
-rw-r----- 1 mysql mysql        257 Aug 19 16:48 server-6-relay-bin.004962
-rw-r----- 1 mysql mysql 1074787881 Aug 19 17:04 server-6-relay-bin.004963
-rw-r----- 1 mysql mysql       3117 Aug 19 17:04 server-6-relay-bin.004964
-rw-r----- 1 mysql mysql        257 Aug 19 17:04 server-6-relay-bin.004965
-rw-r----- 1 mysql mysql 1073742293 Aug 19 17:16 server-6-relay-bin.004966
-rw-r----- 1 mysql mysql        329 Aug 19 17:16 server-6-relay-bin.004967
-rw-r----- 1 mysql mysql        257 Aug 19 17:16 server-6-relay-bin.004968
-rw-r----- 1 mysql mysql 1073875360 Aug 19 17:35 server-6-relay-bin.004969
-rw-r----- 1 mysql mysql     561335 Aug 19 17:35 server-6-relay-bin.004970
-rw-r----- 1 mysql mysql        257 Aug 19 17:35 server-6-relay-bin.004971
-rw-r----- 1 mysql mysql 1075427971 Aug 19 17:51 server-6-relay-bin.004972
-rw-r----- 1 mysql mysql     119238 Aug 19 17:51 server-6-relay-bin.004973
-rw-r----- 1 mysql mysql        257 Aug 19 17:51 server-6-relay-bin.004974
-rw-r----- 1 mysql mysql 1074518543 Aug 19 18:06 server-6-relay-bin.004975
-rw-r----- 1 mysql mysql       1392 Aug 19 18:06 server-6-relay-bin.004976
-rw-r----- 1 mysql mysql        257 Aug 19 18:06 server-6-relay-bin.004977
-rw-r----- 1 mysql mysql 1075430034 Aug 19 18:18 server-6-relay-bin.004978
-rw-r----- 1 mysql mysql        777 Aug 19 18:18 server-6-relay-bin.004979
-rw-r----- 1 mysql mysql        257 Aug 19 18:18 server-6-relay-bin.004980
-rw-r----- 1 mysql mysql 1074768456 Aug 19 18:36 server-6-relay-bin.004981
-rw-r----- 1 mysql mysql        329 Aug 19 18:36 server-6-relay-bin.004982
-rw-r----- 1 mysql mysql        257 Aug 19 18:36 server-6-relay-bin.004983
-rw-r----- 1 mysql mysql 1074375191 Aug 19 18:54 server-6-relay-bin.004984
-rw-r----- 1 mysql mysql        329 Aug 19 18:54 server-6-relay-bin.004985
-rw-r----- 1 mysql mysql        257 Aug 19 18:54 server-6-relay-bin.004986
-rw-r----- 1 mysql mysql 1073742784 Aug 19 19:08 server-6-relay-bin.004987
-rw-r----- 1 mysql mysql       5045 Aug 19 19:08 server-6-relay-bin.004988
-rw-r----- 1 mysql mysql        257 Aug 19 19:08 server-6-relay-bin.004989
-rw-r----- 1 mysql mysql 1074531001 Aug 19 19:24 server-6-relay-bin.004990
-rw-r----- 1 mysql mysql       7092 Aug 19 19:24 server-6-relay-bin.004991
-rw-r----- 1 mysql mysql        257 Aug 19 19:24 server-6-relay-bin.004992
-rw-r----- 1 mysql mysql 1073942849 Aug 19 19:43 server-6-relay-bin.004993
-rw-r----- 1 mysql mysql       3298 Aug 19 19:43 server-6-relay-bin.004994
-rw-r----- 1 mysql mysql        257 Aug 19 19:43 server-6-relay-bin.004995
-rw-r----- 1 mysql mysql 1073747538 Aug 19 20:00 server-6-relay-bin.004996
-rw-r----- 1 mysql mysql     358612 Aug 19 20:00 server-6-relay-bin.004997
-rw-r----- 1 mysql mysql        257 Aug 19 20:00 server-6-relay-bin.004998
-rw-r----- 1 mysql mysql 1075636850 Aug 19 20:12 server-6-relay-bin.004999
-rw-r----- 1 mysql mysql      12810 Aug 19 20:12 server-6-relay-bin.005000
-rw-r----- 1 mysql mysql        257 Aug 19 20:12 server-6-relay-bin.005001
-rw-r----- 1 mysql mysql 1073748897 Aug 19 20:25 server-6-relay-bin.005002
-rw-r----- 1 mysql mysql        329 Aug 19 20:25 server-6-relay-bin.005003
-rw-r----- 1 mysql mysql        257 Aug 19 20:25 server-6-relay-bin.005004
-rw-r----- 1 mysql mysql 1075627773 Aug 19 20:44 server-6-relay-bin.005005
-rw-r----- 1 mysql mysql       1452 Aug 19 20:44 server-6-relay-bin.005006
-rw-r----- 1 mysql mysql        257 Aug 19 20:44 server-6-relay-bin.005007
-rw-r----- 1 mysql mysql 1073973063 Aug 19 21:02 server-6-relay-bin.005008
-rw-r----- 1 mysql mysql      12392 Aug 19 21:02 server-6-relay-bin.005009
-rw-r----- 1 mysql mysql        257 Aug 19 21:02 server-6-relay-bin.005010
-rw-r----- 1 mysql mysql 1074198559 Aug 19 21:18 server-6-relay-bin.005011
-rw-r----- 1 mysql mysql       9813 Aug 19 21:18 server-6-relay-bin.005012
-rw-r----- 1 mysql mysql        257 Aug 19 21:18 server-6-relay-bin.005013
-rw-r----- 1 mysql mysql 1073750958 Aug 19 21:35 server-6-relay-bin.005014
-rw-r----- 1 mysql mysql     415417 Aug 19 21:35 server-6-relay-bin.005015
-rw-r----- 1 mysql mysql        257 Aug 19 21:35 server-6-relay-bin.005016
-rw-r----- 1 mysql mysql 1073757766 Aug 19 21:55 server-6-relay-bin.005017
-rw-r----- 1 mysql mysql     484712 Aug 19 21:55 server-6-relay-bin.005018
-rw-r----- 1 mysql mysql        257 Aug 19 21:55 server-6-relay-bin.005019
-rw-r----- 1 mysql mysql 1074704096 Aug 19 22:14 server-6-relay-bin.005020
-rw-r----- 1 mysql mysql        329 Aug 19 22:14 server-6-relay-bin.005021
-rw-r----- 1 mysql mysql        257 Aug 19 22:14 server-6-relay-bin.005022
-rw-r----- 1 mysql mysql 1074567071 Aug 19 22:25 server-6-relay-bin.005023
-rw-r----- 1 mysql mysql       3299 Aug 19 22:25 server-6-relay-bin.005024
-rw-r----- 1 mysql mysql        257 Aug 19 22:25 server-6-relay-bin.005025
-rw-r----- 1 mysql mysql 1073747930 Aug 19 22:35 server-6-relay-bin.005026
-rw-r----- 1 mysql mysql     152479 Aug 19 22:35 server-6-relay-bin.005027
-rw-r----- 1 mysql mysql        257 Aug 19 22:35 server-6-relay-bin.005028
-rw-r----- 1 mysql mysql 1073749573 Aug 19 22:55 server-6-relay-bin.005029
-rw-r----- 1 mysql mysql     206060 Aug 19 22:55 server-6-relay-bin.005030
-rw-r----- 1 mysql mysql        257 Aug 19 22:55 server-6-relay-bin.005031
-rw-r----- 1 mysql mysql 1073974611 Aug 19 23:14 server-6-relay-bin.005032
-rw-r----- 1 mysql mysql      16223 Aug 19 23:14 server-6-relay-bin.005033
-rw-r----- 1 mysql mysql        257 Aug 19 23:14 server-6-relay-bin.005034
-rw-r----- 1 mysql mysql 1073747289 Aug 19 23:30 server-6-relay-bin.005035
-rw-r----- 1 mysql mysql     394623 Aug 19 23:30 server-6-relay-bin.005036
-rw-r----- 1 mysql mysql        257 Aug 19 23:30 server-6-relay-bin.005037
-rw-r----- 1 mysql mysql 1073964634 Aug 19 23:40 server-6-relay-bin.005038
-rw-r----- 1 mysql mysql        329 Aug 19 23:40 server-6-relay-bin.005039
-rw-r----- 1 mysql mysql        257 Aug 19 23:40 server-6-relay-bin.005040
-rw-r----- 1 mysql mysql 1074362909 Aug 19 23:51 server-6-relay-bin.005041
-rw-r----- 1 mysql mysql       3074 Aug 19 23:51 server-6-relay-bin.005042
-rw-r----- 1 mysql mysql        257 Aug 19 23:51 server-6-relay-bin.005043
-rw-r----- 1 mysql mysql 1073753748 Aug 20 00:06 server-6-relay-bin.005044
-rw-r----- 1 mysql mysql     365716 Aug 20 00:06 server-6-relay-bin.005045
-rw-r----- 1 mysql mysql        257 Aug 20 00:06 server-6-relay-bin.005046
-rw-r----- 1 mysql mysql 1075420004 Aug 20 00:21 server-6-relay-bin.005047
-rw-r----- 1 mysql mysql       4027 Aug 20 00:21 server-6-relay-bin.005048
-rw-r----- 1 mysql mysql        257 Aug 20 00:21 server-6-relay-bin.005049
-rw-r----- 1 mysql mysql 1073757176 Aug 20 00:40 server-6-relay-bin.005050
-rw-r----- 1 mysql mysql      17029 Aug 20 00:40 server-6-relay-bin.005051
-rw-r----- 1 mysql mysql        257 Aug 20 00:40 server-6-relay-bin.005052
-rw-r----- 1 mysql mysql 1074848048 Aug 20 00:56 server-6-relay-bin.005053
-rw-r----- 1 mysql mysql       2106 Aug 20 00:56 server-6-relay-bin.005054
-rw-r----- 1 mysql mysql        257 Aug 20 00:56 server-6-relay-bin.005055
-rw-r----- 1 mysql mysql 1073744109 Aug 20 01:15 server-6-relay-bin.005056
-rw-r----- 1 mysql mysql     167283 Aug 20 01:15 server-6-relay-bin.005057
-rw-r----- 1 mysql mysql        257 Aug 20 01:15 server-6-relay-bin.005058
-rw-r----- 1 mysql mysql 1073863153 Aug 20 01:31 server-6-relay-bin.005059
-rw-r----- 1 mysql mysql        832 Aug 20 01:31 server-6-relay-bin.005060
-rw-r----- 1 mysql mysql        257 Aug 20 01:31 server-6-relay-bin.005061
-rw-r----- 1 mysql mysql 1073755275 Aug 20 01:50 server-6-relay-bin.005062
-rw-r----- 1 mysql mysql     351680 Aug 20 01:50 server-6-relay-bin.005063
-rw-r----- 1 mysql mysql        257 Aug 20 01:50 server-6-relay-bin.005064
-rw-r----- 1 mysql mysql 1073745924 Aug 20 02:00 server-6-relay-bin.005065
-rw-r----- 1 mysql mysql     184126 Aug 20 02:00 server-6-relay-bin.005066
-rw-r----- 1 mysql mysql        257 Aug 20 02:00 server-6-relay-bin.005067
-rw-r----- 1 mysql mysql 1074559080 Aug 20 02:11 server-6-relay-bin.005068
-rw-r----- 1 mysql mysql        832 Aug 20 02:11 server-6-relay-bin.005069
-rw-r----- 1 mysql mysql        257 Aug 20 02:11 server-6-relay-bin.005070
-rw-r----- 1 mysql mysql 1074703077 Aug 20 02:21 server-6-relay-bin.005071
-rw-r----- 1 mysql mysql        832 Aug 20 02:21 server-6-relay-bin.005072
-rw-r----- 1 mysql mysql        257 Aug 20 02:21 server-6-relay-bin.005073
-rw-r----- 1 mysql mysql 1075168834 Aug 20 02:31 server-6-relay-bin.005074
-rw-r----- 1 mysql mysql       3009 Aug 20 02:31 server-6-relay-bin.005075
-rw-r----- 1 mysql mysql        257 Aug 20 02:31 server-6-relay-bin.005076
-rw-r----- 1 mysql mysql 1073786213 Aug 20 02:41 server-6-relay-bin.005077
-rw-r----- 1 mysql mysql       1969 Aug 20 02:41 server-6-relay-bin.005078
-rw-r----- 1 mysql mysql        257 Aug 20 02:41 server-6-relay-bin.005079
-rw-r----- 1 mysql mysql 1075261516 Aug 20 02:52 server-6-relay-bin.005080
-rw-r----- 1 mysql mysql        329 Aug 20 02:52 server-6-relay-bin.005081
-rw-r----- 1 mysql mysql        257 Aug 20 02:52 server-6-relay-bin.005082
-rw-r----- 1 mysql mysql 1074090288 Aug 20 03:03 server-6-relay-bin.005083
-rw-r----- 1 mysql mysql        329 Aug 20 03:03 server-6-relay-bin.005084
-rw-r----- 1 mysql mysql        257 Aug 20 03:03 server-6-relay-bin.005085
-rw-r----- 1 mysql mysql 1074140925 Aug 20 03:13 server-6-relay-bin.005086
-rw-r----- 1 mysql mysql        329 Aug 20 03:13 server-6-relay-bin.005087
-rw-r----- 1 mysql mysql        257 Aug 20 03:13 server-6-relay-bin.005088
-rw-r----- 1 mysql mysql 1074942298 Aug 20 03:23 server-6-relay-bin.005089
-rw-r----- 1 mysql mysql        329 Aug 20 03:23 server-6-relay-bin.005090
-rw-r----- 1 mysql mysql        257 Aug 20 03:23 server-6-relay-bin.005091
-rw-r----- 1 mysql mysql 1073946866 Aug 20 03:33 server-6-relay-bin.005092
-rw-r----- 1 mysql mysql        329 Aug 20 03:33 server-6-relay-bin.005093
-rw-r----- 1 mysql mysql        257 Aug 20 03:33 server-6-relay-bin.005094
-rw-r----- 1 mysql mysql 1074995326 Aug 20 03:43 server-6-relay-bin.005095
-rw-r----- 1 mysql mysql        329 Aug 20 03:43 server-6-relay-bin.005096
-rw-r----- 1 mysql mysql        257 Aug 20 03:43 server-6-relay-bin.005097
-rw-r----- 1 mysql mysql 1074411312 Aug 20 03:53 server-6-relay-bin.005098
-rw-r----- 1 mysql mysql        329 Aug 20 03:53 server-6-relay-bin.005099
-rw-r----- 1 mysql mysql        257 Aug 20 03:53 server-6-relay-bin.005100
-rw-r----- 1 mysql mysql 1073961985 Aug 20 04:03 server-6-relay-bin.005101
-rw-r----- 1 mysql mysql       1581 Aug 20 04:03 server-6-relay-bin.005102
-rw-r----- 1 mysql mysql        257 Aug 20 04:03 server-6-relay-bin.005103
-rw-r----- 1 mysql mysql 1073987949 Aug 20 04:14 server-6-relay-bin.005104
-rw-r----- 1 mysql mysql        329 Aug 20 04:14 server-6-relay-bin.005105
-rw-r----- 1 mysql mysql        257 Aug 20 04:14 server-6-relay-bin.005106
-rw-r----- 1 mysql mysql 1074580980 Aug 20 04:23 server-6-relay-bin.005107
-rw-r----- 1 mysql mysql        329 Aug 20 04:23 server-6-relay-bin.005108
-rw-r----- 1 mysql mysql        257 Aug 20 04:23 server-6-relay-bin.005109
-rw-r----- 1 mysql mysql 1074915235 Aug 20 04:32 server-6-relay-bin.005110
-rw-r----- 1 mysql mysql        329 Aug 20 04:32 server-6-relay-bin.005111
-rw-r----- 1 mysql mysql        257 Aug 20 04:32 server-6-relay-bin.005112
-rw-r----- 1 mysql mysql 1073750177 Aug 20 04:42 server-6-relay-bin.005113
-rw-r----- 1 mysql mysql        329 Aug 20 04:42 server-6-relay-bin.005114
-rw-r----- 1 mysql mysql        257 Aug 20 04:42 server-6-relay-bin.005115
-rw-r----- 1 mysql mysql 1075582907 Aug 20 04:52 server-6-relay-bin.005116
-rw-r----- 1 mysql mysql       1654 Aug 20 04:52 server-6-relay-bin.005117
-rw-r----- 1 mysql mysql        257 Aug 20 04:52 server-6-relay-bin.005118
-rw-r----- 1 mysql mysql 1073831777 Aug 20 05:02 server-6-relay-bin.005119
-rw-r----- 1 mysql mysql        944 Aug 20 05:02 server-6-relay-bin.005120
-rw-r----- 1 mysql mysql        257 Aug 20 05:02 server-6-relay-bin.005121
-rw-r----- 1 mysql mysql 1073939965 Aug 20 05:12 server-6-relay-bin.005122
-rw-r----- 1 mysql mysql        329 Aug 20 05:12 server-6-relay-bin.005123
-rw-r----- 1 mysql mysql        257 Aug 20 05:12 server-6-relay-bin.005124
-rw-r----- 1 mysql mysql 1074975635 Aug 20 05:23 server-6-relay-bin.005125
-rw-r----- 1 mysql mysql        329 Aug 20 05:23 server-6-relay-bin.005126
-rw-r----- 1 mysql mysql        257 Aug 20 05:23 server-6-relay-bin.005127
-rw-r----- 1 mysql mysql 1073775752 Aug 20 05:34 server-6-relay-bin.005128
-rw-r----- 1 mysql mysql        329 Aug 20 05:34 server-6-relay-bin.005129
-rw-r----- 1 mysql mysql        257 Aug 20 05:34 server-6-relay-bin.005130
-rw-r----- 1 mysql mysql 1073748439 Aug 20 05:45 server-6-relay-bin.005131
-rw-r----- 1 mysql mysql      50123 Aug 20 05:45 server-6-relay-bin.005132
-rw-r----- 1 mysql mysql        257 Aug 20 05:45 server-6-relay-bin.005133
-rw-r----- 1 mysql mysql 1074232059 Aug 20 06:00 server-6-relay-bin.005134
-rw-r----- 1 mysql mysql     218243 Aug 20 06:00 server-6-relay-bin.005135
-rw-r----- 1 mysql mysql        257 Aug 20 06:00 server-6-relay-bin.005136
-rw-r----- 1 mysql mysql 1073742109 Aug 20 06:29 server-6-relay-bin.005137
-rw-r----- 1 mysql mysql     566273 Aug 20 06:29 server-6-relay-bin.005138
-rw-r----- 1 mysql mysql        257 Aug 20 06:29 server-6-relay-bin.005139
-rw-r----- 1 mysql mysql 1073743811 Aug 20 06:50 server-6-relay-bin.005140
-rw-r----- 1 mysql mysql     496397 Aug 20 06:50 server-6-relay-bin.005141
-rw-r----- 1 mysql mysql        257 Aug 20 06:50 server-6-relay-bin.005142
-rw-r----- 1 mysql mysql 1074448682 Aug 20 07:04 server-6-relay-bin.005143
-rw-r----- 1 mysql mysql        329 Aug 20 07:04 server-6-relay-bin.005144
-rw-r----- 1 mysql mysql        257 Aug 20 07:04 server-6-relay-bin.005145
-rw-r----- 1 mysql mysql 1073933474 Aug 20 07:16 server-6-relay-bin.005146
-rw-r----- 1 mysql mysql       5355 Aug 20 07:16 server-6-relay-bin.005147
-rw-r----- 1 mysql mysql        257 Aug 20 07:16 server-6-relay-bin.005148
-rw-r----- 1 mysql mysql 1074128415 Aug 20 07:29 server-6-relay-bin.005149
-rw-r----- 1 mysql mysql        832 Aug 20 07:29 server-6-relay-bin.005150
-rw-r----- 1 mysql mysql        257 Aug 20 07:29 server-6-relay-bin.005151
-rw-r----- 1 mysql mysql 1074672741 Aug 20 07:40 server-6-relay-bin.005152
-rw-r----- 1 mysql mysql       1335 Aug 20 07:40 server-6-relay-bin.005153
-rw-r----- 1 mysql mysql        257 Aug 20 07:40 server-6-relay-bin.005154
-rw-r----- 1 mysql mysql 1074458702 Aug 20 07:51 server-6-relay-bin.005155
-rw-r----- 1 mysql mysql      10856 Aug 20 07:51 server-6-relay-bin.005156
-rw-r----- 1 mysql mysql        257 Aug 20 07:51 server-6-relay-bin.005157
-rw-r----- 1 mysql mysql 1073924438 Aug 20 08:01 server-6-relay-bin.005158
-rw-r----- 1 mysql mysql        832 Aug 20 08:01 server-6-relay-bin.005159
-rw-r----- 1 mysql mysql        257 Aug 20 08:01 server-6-relay-bin.005160
-rw-r----- 1 mysql mysql 1075303853 Aug 20 08:13 server-6-relay-bin.005161
-rw-r----- 1 mysql mysql       3783 Aug 20 08:13 server-6-relay-bin.005162
-rw-r----- 1 mysql mysql        257 Aug 20 08:13 server-6-relay-bin.005163
-rw-r----- 1 mysql mysql 1075011674 Aug 20 08:25 server-6-relay-bin.005164
-rw-r----- 1 mysql mysql        329 Aug 20 08:25 server-6-relay-bin.005165
-rw-r----- 1 mysql mysql        257 Aug 20 08:25 server-6-relay-bin.005166
-rw-r----- 1 mysql mysql 1073759149 Aug 20 08:45 server-6-relay-bin.005167
-rw-r----- 1 mysql mysql     595560 Aug 20 08:45 server-6-relay-bin.005168
-rw-r----- 1 mysql mysql        257 Aug 20 08:45 server-6-relay-bin.005169
-rw-r----- 1 mysql mysql 1073744454 Aug 20 09:21 server-6-relay-bin.005170
-rw-r----- 1 mysql mysql        329 Aug 20 09:21 server-6-relay-bin.005171
-rw-r----- 1 mysql mysql        257 Aug 20 09:21 server-6-relay-bin.005172
-rw-r----- 1 mysql mysql 1073744721 Aug 20 09:59 server-6-relay-bin.005173
-rw-r----- 1 mysql mysql        329 Aug 20 09:59 server-6-relay-bin.005174
-rw-r----- 1 mysql mysql        257 Aug 20 09:59 server-6-relay-bin.005175
-rw-r----- 1 mysql mysql 1073744402 Aug 20 10:33 server-6-relay-bin.005176
-rw-r----- 1 mysql mysql       1415 Aug 20 10:33 server-6-relay-bin.005177
-rw-r----- 1 mysql mysql        257 Aug 20 10:33 server-6-relay-bin.005178
-rw-r----- 1 mysql mysql  492892939 Aug 20 10:47 server-6-relay-bin.005179
-rw-r----- 1 mysql mysql        210 Aug 20 10:47 server-6-relay-bin.005180
-rw-r----- 1 mysql mysql  581257657 Aug 20 11:05 server-6-relay-bin.005181
-rw-r----- 1 mysql mysql        257 Aug 20 11:05 server-6-relay-bin.005182
-rw-r----- 1 mysql mysql   89230476 Aug 20 11:07 server-6-relay-bin.005183
```





4.查看innoDB的相关配置:

```
mysql> show global status like '%innodb_log%';
+---------------------------+------------+
| Variable_name             | Value      |
+---------------------------+------------+
| Innodb_log_waits          | 0          |
| Innodb_log_write_requests | 1563734583 |
| Innodb_log_writes         | 259641866  |
+---------------------------+------------+
3 rows in set (0.01 sec)

mysql> show global status like '%innodb_buffer_pool_wait%';
+------------------------------+----------+
| Variable_name                | Value    |
+------------------------------+----------+
| Innodb_buffer_pool_wait_free | 13307693 |
+------------------------------+----------+
1 row in set (0.00 sec)
```

发现缓冲池有大量的空闲页等待被执行



5.初步怀疑是从库的sql语句读写速度比较慢,查看服务器磁盘IO情况:

```
[root@server-6 ~]# iostat -d -x 2
Linux 3.10.0-693.el7.x86_64 (server-6) 	08/20/2018 	_x86_64_	(56 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               1.06    14.14   80.26 1964.48  6209.75 19539.84    25.19     0.79    0.38   10.58    0.47   0.18  37.16
dm-0              0.00     0.00   80.18 1977.08  6205.01 19533.60    25.02     0.07    0.04    0.45    0.02   0.18  37.17
dm-1              0.00     0.00    1.18    1.56     4.74     6.23     8.00     0.03   11.99    4.65   17.57   0.12   0.03

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00    86.50  294.50 2278.00 22820.00 38805.75    47.91    19.72    7.54   58.32    0.98   0.39 100.00
dm-0              0.00     0.00  296.50 2327.00 22852.00 38221.75    46.56    20.08    7.54   57.92    1.12   0.38  99.95
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00    31.50  497.00 2887.00 38196.00 42797.50    47.87    14.84    4.59   29.85    0.24   0.30 100.00
dm-0              0.00     0.00  491.50 2918.50 37828.00 42797.50    47.29    14.88    4.56   30.19    0.25   0.29 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00  433.50 2135.00 34016.00 61046.25    74.02    11.60    3.98   17.68    1.20   0.39  99.85
dm-0              0.00     0.00  439.50 2136.00 34680.00 61052.25    74.34    11.61    3.97   17.43    1.20   0.39  99.90
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00  828.00 1987.00 61214.00 21236.25    58.58    11.50    4.56   15.08    0.17   0.36  99.95
dm-0              0.00     0.00  824.00 1986.00 60774.00 21230.25    58.37    11.52    4.57   15.15    0.18   0.36 100.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

```

磁盘IO负载已经达到了100%.

6.查看占用磁盘IO的主要进程:

```
Total DISK READ :      25.90 M/s | Total DISK WRITE :	   64.71 M/s
Actual DISK READ:      27.85 M/s | Actual DISK WRITE:	   52.46 M/s
   TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
 93677 be/4 mysql    1280.78 K/s   54.50 K/s  0.00 % 89.19 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
124925 be/4 mysql       5.30 M/s  735.77 K/s  0.00 % 86.79 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 86834 be/4 mysql    1853.04 K/s  711.92 K/s  0.00 % 85.62 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
124935 be/4 mysql       2.83 M/s  817.52 K/s  0.00 % 77.32 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
127362 be/4 mysql    1825.79 K/s  545.01 K/s  0.00 % 74.37 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
124928 be/4 mysql       3.94 M/s  517.76 K/s  0.00 % 74.02 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 31008 be/4 mysql       2.20 M/s  531.39 K/s  0.00 % 72.05 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 30965 be/4 mysql     572.26 K/s  180.54 K/s  0.00 % 68.54 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 54715 be/4 mysql       3.67 M/s 1062.77 K/s  0.00 % 67.73 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 30967 be/4 mysql     545.01 K/s  953.77 K/s  0.00 % 57.04 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 30966 be/4 mysql    1021.90 K/s 1086.62 K/s  0.00 % 44.76 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 30968 be/4 mysql     803.89 K/s  909.49 K/s  0.00 % 33.53 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 23438 be/4 mysql     136.25 K/s 1216.06 K/s  0.00 %  4.78 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 30948 be/4 mysql       0.00 B/s    0.00 B/s  0.00 %  3.82 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 30945 be/4 mysql       0.00 B/s    0.00 B/s  0.00 %  1.16 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid
 30949 be/4 mysql       0.00 B/s   36.62 M/s  0.00 %  0.42 % mysqld --daemonize --pid-file=/var/run/mysqld/mysqld.pid

```



7.查看mysql有关磁盘IO的主要配置参数:

```
mysql> show variables like '%sync_bin%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| sync_binlog   | 1     |
+---------------+-------+
1 row in set (0.00 sec)

mysql> show variables like '%innodb_flush%';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_timeout    | 1     |
| innodb_flush_log_at_trx_commit | 1     |
| innodb_flush_method            |       |
| innodb_flush_neighbors         | 1     |
| innodb_flush_sync              | ON    |
| innodb_flushing_avg_loops      | 30    |
+--------------------------------+-------+
6 rows in set (0.00 sec)

```

sync_binlog=1表示每次事务提交后MySQL将进行一次fsync之类的磁盘同步指令来将binlog_cache中的数据强制写入磁盘，频繁的写盘导致磁盘IO居高不下 

innodb_flush_log_at_trx_commit=1时,log buffer 会被写入到日志文件并刷写到磁盘。这也是默认值。这是最安全的配置，但由于每次事务都需要进行磁盘I/O，所以也最慢 

有关这两个参数的更多解释,请百度



8.修改参数配置:

```
编辑my.cnf配置文件:
[root@server-6 ~]# vim /etc/my.cnf
sync_binlog=0
innodb_flush_log_at_trx_commit=2
```



9.重启mysql服务,重启slave进程

```
[root@server-6 ~]# service mysqld restart

[root@server-6 ~]# mysql -uroot -p
Enter password:
mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```



10.查看参数是否已经生效:

```
mysql> show variables like '%sync_bin%';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| sync_binlog   | 0     |
+---------------+-------+
1 row in set (0.01 sec)

mysql> show variables like '%innodb_flush%';
+--------------------------------+-------+
| Variable_name                  | Value |
+--------------------------------+-------+
| innodb_flush_log_at_timeout    | 1     |
| innodb_flush_log_at_trx_commit | 2     |
| innodb_flush_method            |       |
| innodb_flush_neighbors         | 1     |
| innodb_flush_sync              | ON    |
| innodb_flushing_avg_loops      | 30    |
+--------------------------------+-------+
6 rows in set (0.00 sec)
```



11.但是查看磁盘IO还是居高不下

```
[root@server-6 mysql]# iostat -d -x 2
Linux 3.10.0-693.el7.x86_64 (server-6) 	08/20/2018 	_x86_64_	(56 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               1.06    14.18   80.43 1966.48  6216.32 19621.73    25.25     0.80    0.39   10.68    0.47   0.18  37.31
dm-0              0.00     0.00   80.35 1979.12  6211.59 19615.51    25.08     0.09    0.04    0.60    0.02   0.18  37.33
dm-1              0.00     0.00    1.18    1.55     4.73     6.22     8.00     0.03   11.99    4.65   17.57   0.12   0.03

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00   44.50 3126.00  1534.00 67250.50    43.39     3.18    1.10   51.88    0.38   0.31  96.95
dm-0              0.00     0.00   44.00 3126.50  1526.00 67258.50    43.39     3.17    1.10   52.47    0.37   0.31  96.75
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00    7.50 3152.50   600.00 51644.00    33.07     2.69    0.87   81.47    0.68   0.30  94.10
dm-0              0.00     0.00    7.50 3152.00   600.00 51636.00    33.07     2.69    0.87   81.47    0.67   0.30  94.00
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00   30.00 2915.00  1334.00 53757.25    37.41     3.74    1.27   36.00    0.91   0.30  88.95
dm-0              0.00     0.00   30.50 2915.00  1398.00 53757.25    37.45     3.73    1.27   35.43    0.91   0.30  88.80
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               0.00     0.00   66.50 3325.00  1656.00 65172.75    39.41     3.91    1.15   46.50    0.24   0.29  99.15
dm-0              0.00     0.00   66.50 3325.00  1600.00 65172.75    39.38     3.90    1.14   46.50    0.24   0.29  99.10
dm-1              0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00    0.00    0.00   0.00   0.00
```

此外,延时仍然在不断的上升,

```
[root@server-6 mysql]# /usr/bin/pt-heartbeat --host=127.0.0.1 --user=heartbeat --password=xxxxxx -D dwd_analystic  --master-server-id=63306  --check
94110.00
```



12.但是查看slave状态.仍然在不停的读取bin日志和log信息.但是读取速度非常慢

```
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.81.61.101
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.005029
          Read_Master_Log_Pos: 195990517
               Relay_Log_File: server-6-relay-bin.004900
                Relay_Log_Pos: 983195495
        Relay_Master_Log_File: mysql-bin.004930
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: dwd_analystic,hsq_sync_RDS
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 983195282
              Relay_Log_Space: 106561330708
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 94159
```



13.查看mysql数据文件的binlog日志文件个数

```
[root@server-6 mysql]# ls server-6-relay-bin.* | wc -l
301
```

---

14.优化完sync磁盘写入机制后发现延时还是非常高,而且在不断上升,最高的时候达到了34个小时.slave读取binlog日志还是非常慢.  

查看一下mysql数据库的关键性能配置参数,发现没有经过任何优化.这极大的限制了mysql的性能.

编辑my.cnf配置文件

```
sync_binlog=0
innodb_flush_log_at_trx_commit=2
innodb_buffer_pool_size=100G
innodb_page_cleaners=2
innodb_log_file_size=1G
```

重启mysql服务,启动slave进程

```
service mysqld restart
mysql -uroot -p
mysql> start slave;
```


下面是修改过后的部分配置参数:

```
#下面的参数是最关键的性能指标.默认值是148M.一般建议是服务器内存的70%-75%
由于我服务器是189G的内存,考虑到其他耗内存的进程,所以设置为100G

mysql> show variables like "innodb_buffer_pool_size%";
+-------------------------+--------------+
| Variable_name           | Value        |
+-------------------------+--------------+
| innodb_buffer_pool_size | 107374182400 |
+-------------------------+--------------+
1 row in set (0.00 sec)

#Instances参数和上面的pool_size相关联.这个是自动设置的.

mysql> show variables like "innodb_buffer_pool_instances";
+------------------------------+-------+
| Variable_name                | Value |
+------------------------------+-------+
| innodb_buffer_pool_instances | 8     |
+------------------------------+-------+
1 row in set (0.00 sec)

#下面是日志的参数.log_file_size定义了日志文件的大小,这个值调高为好

mysql> show variables like "innodb_log_file%";
+---------------------------+------------+
| Variable_name             | Value      |
+---------------------------+------------+
| innodb_log_file_size      | 1073741824 |
| innodb_log_files_in_group | 2          |
+---------------------------+------------+
2 rows in set (0.00 sec)

```
有关mysql的参数的配置和优化.请参考mysql的官方文档:[Server Option, System Variable, and Status Variable Reference](https://dev.mysql.com/doc/refman/5.7/en/server-option-variable-reference.html)

---

此时,虽然延时还在上涨.但是slave读取binlog日志的速度明显快了很多,相比之前同步速度几乎是指数级的翻倍上涨.性能大幅提升.而且从库上的binlog日志文件在不断减少.
```
Binlog文件从300多个在逐渐减少:
[root@server-6 ~]# ls /data/mysql/server-6-relay-bin.*  | wc -l
243
```

观察了1天之后,日志文件进一步减少,而且延时在不断的最近

```
[root@server-6 ~]# /usr/bin/pt-heartbeat --host=127.0.0.1 --user=heartbeat --password=xxxx -D dwd_analystic  --master-server-id=63306  --check
124563.00
[root@server-6 ~]# /usr/bin/pt-heartbeat --host=127.0.0.1 --user=heartbeat --password=xxxx -D dwd_analystic  --master-server-id=63306  --check
124547.00
[root@server-6 ~]# /usr/bin/pt-heartbeat --host=127.0.0.1 --user=heartbeat --password=xxxx -D dwd_analystic  --master-server-id=63306  --check
124532.00
[root@server-6 ~]# ls /data/mysql/server-6-relay-bin.*  | wc -l
183
```

再观察一个晚上后,延时故障消失,主从恢复正常.binlog文件几乎读完了.

```
[root@server-6 ~]# ls /data/mysql/server-6-relay-bin.*  | wc -l
3
[root@server-6 ~]# /usr/bin/pt-heartbeat --host=127.0.0.1 --user=heartbeat --password=xxxx -D dwd_analystic  --master-server-id=63306  --check
1.00
```

奇怪的是磁盘的IO负载还是非常高:

```
[root@server-6 ~]# iostat -d -x 2
Linux 3.10.0-693.el7.x86_64 (server-6) 	08/22/2018 	_x86_64_	(56 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda               3.05    18.97   82.57 1959.78  6145.11 20097.05    25.70     0.02    0.01    0.10    0.00   0.19  39.06
dm-0              0.00     0.00   82.12 1972.87  6130.89 20073.49    25.50     0.27    0.13    1.96    0.05   0.19  38.91
dm-1              0.00     0.00    3.55    5.89    14.21    23.56     8.00     0.23   24.24   22.84   25.08   0.79   0.74

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              55.50    72.50  218.50 1409.50  1272.00 12573.00    17.01     3.45    1.91   13.30    0.15   0.61  99.85
dm-0              0.00     0.00  207.00 1482.00  1018.00 12573.00    16.09     3.25    1.73   12.88    0.18   0.59  99.70
dm-1              0.00     0.00   68.50    0.00   274.00     0.00     8.00     1.21   16.61   16.61    0.00   2.69  18.40

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              51.50     0.00  144.50 1809.00  1120.00 12491.25    13.94     2.95    1.69   19.88    0.24   0.51  99.80
dm-0              0.00     0.00  136.50 1809.00   866.00 12491.25    13.73     2.63    1.53   18.63    0.24   0.51  99.70
dm-1              0.00     0.00   58.50    0.00   234.00     0.00     8.00     2.47   43.60   43.60    0.00   5.34  31.25

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sda              46.00    67.00  186.00 1726.00  1354.00 13791.75    15.84     2.73    1.25   11.63    0.13   0.52  99.50
dm-0              0.00     0.00  179.50 1793.00  1158.00 13791.75    15.16     2.49    1.15   11.18    0.15   0.50  99.45
dm-1              0.00     0.00   53.50    0.00   214.00     0.00     8.00     2.14   21.75   21.75    0.00   4.38  23.45
```

但是磁盘的读写速度并不高,而且是es和kafka进程在消耗磁盘IO.并不是mysql进程了

```
Total DISK READ :     131.73 K/s | Total DISK WRITE :	   11.93 M/s
Actual DISK READ:     185.77 K/s | Actual DISK WRITE:	   16.11 M/s
   TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO>    COMMAND
   305 be/4 root        0.00 B/s    0.00 B/s  0.00 % 88.69 % [kswapd0]
117157 be/4 hadoop     50.66 K/s    3.38 K/s  0.00 %  8.19 % java -Xmx64G -Xms64G -server -XX:+UseG1GC -~a.Kafka /opt/kafka/config/server.properties
140979 be/4 hadoop      0.00 B/s  118.22 K/s  0.00 %  6.71 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
140960 be/4 hadoop      0.00 B/s   50.66 K/s  0.00 %  0.38 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
140449 be/4 hadoop      0.00 B/s  148.61 K/s  0.00 %  0.26 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
140974 be/4 hadoop      0.00 B/s  138.48 K/s  0.00 %  0.26 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
140980 be/4 hadoop      0.00 B/s  141.86 K/s  0.00 %  0.25 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
140973 be/4 hadoop      0.00 B/s  145.24 K/s  0.00 %  0.24 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
140869 be/4 hadoop      0.00 B/s   43.91 K/s  0.00 %  0.23 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
140966 be/4 hadoop      0.00 B/s  179.01 K/s  0.00 %  0.22 % java -Xms24g -Xmx24g -XX:+UseConcMarkSweepG~rg.elasticsearch.bootstrap.Elasticsearch -d
```
---

总结: 

主从延时比较关键的地方:

1.确保slave库的服务器配置和master相当,或者比master更高.    
2.确保网络延时较低.      
3.sync_binlog和innodb_flush_log_at_trx_commit,这两个参数决定了磁盘写入机制.如果sync_binlog设置为1.则每次操作都要回写到磁盘日志,极大的增加磁盘IO负载和同步负担.    
4.innodb_buffer_pool_size参数调整会将mysql的整体性能提高到好几个倍数.    
5.如果是mysql5.7以上版本,尽量使用GTID主从复制机制代替传统的Binlog机制.因为Binlog的sql线程还是单线程工作模式.    
6.完善的监控机制.如果第一时间发现延时较高,就要尽早介入处理.
