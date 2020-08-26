---
title:  ELK收集mysql5.7慢日志
date: 2020-08-25 22:59:58
tags:  elk
categories: elk
comments: true
copyright: true
---



## ELK收集mysql5.7慢日志

### 介绍

公司ELK平台计划收集生产业务的所有mysql慢日志.由于所有环境中均使用mysql5.7版本,所以其他mysql版本的慢日志格式不在讨论范围之内.

慢日志的grok正则匹配我折腾了很久,网上的大多文档中给出的logstash的grok正则其实并不能正确的解析到mysql慢日志的字段.

这个博客的grok正则经过实践可行.而且filebeat,logstash的filter配置也是参考这个博客配置的:[博客地址](https://www.cnblogs.com/minseo/p/10441913.html)

---

### MySQL慢日志

慢日志格式如下:

```
[work@msf-mysql-master log]$ sudo head slow_2019072103.log
/usr/local/mysql5.7/bin/mysqld, Version: 5.7.24-log (MySQL Community Server (GPL)). started with:
Tcp port: 3306  Unix socket: /mysql_log/msf/tmp/mysql.sock
Time                 Id Command    Argument
# Time: 2019-07-21T08:54:04.145255+08:00
# User@Host: u_msf[u_msf] @  [10.111.10.40]  Id: 131421254
# Query_time: 1.595300  Lock_time: 0.000031 Rows_sent: 20  Rows_examined: 809259
use msf_prod;
SET timestamp=1563670444;
SELECT `id`,`type`,`honey`,`remark`,`created_at` FROM `t_log_user_honey` WHERE `user_id` = '1000014423' ORDER BY `created_at` DESC LIMIT 20 OFFSET 0;
# Time: 2019-07-21T10:51:06.184010+08:00
```

<!--more-->

每个日志文件的格式为`slow_日期.log` .7天切割一次新的日志文件

每个日志的开头三行是不需要的内容,所以需要filebeat排除

每一条慢日志有以下几行组成:

```
# Time: 2019-07-21T08:54:04.145255+08:00
# User@Host: u_msf[u_msf] @  [10.111.10.40]  Id: 131421254
# Query_time: 1.595300  Lock_time: 0.000031 Rows_sent: 20  Rows_examined: 809259
use msf_prod;
SET timestamp=1563670444;
SELECT `id`,`type`,`honey`,`remark`,`created_at` FROM `t_log_user_honey` WHERE `user_id` = '1000014423' ORDER BY `created_at` DESC LIMIT 20 OFFSET 0;
```

第一行Time时间不需要,所以也需要filebeat排除.

从第二行开始匹配,有些慢日志可能没有`use database;`的语句.所以需要分别针对对待

---

### filebeat配置

filebeat需要开启多行日志功能.并且排除特定的字段.除此之外,和其他的日志收集配置一样.下面是生产环境中filebeat的配置

```
- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /mysql_log/msf/log/slow_*.log
  exclude_lines: ['^\# Time','^\/usr','^Tcp','^Time']
  multiline.pattern: '^\# Time|^\# User'
  multiline.negate: true
  multiline.match: after
  fields:
    project: msf
    type: mysql
    level: slow
```

---

### logstash配置

logstash需要使用正则匹配2种格式的慢日志.当一种grok匹配到了后,logstash就不会再接着往下匹配了,所以每条日志只会匹配一种grok规则

```
      if [fields][type] == "mysql" {
          grok {
            #有use database语句
            match => [ "message" , "^#\s+User@Host:\s+%{USER:user}\[[^\]]+\]\s+@\s+(?:(?<clienthost>\S*) )?\[(?:%{IPV4:clientip})?\]\s+Id:\s+%{NUMBER:row_id:int}\n#\s+Query_time:\s+%{NUMBER:query_time:float}\s+Lock_time:\s+%{NUMBER:lock_time:float}\s+Rows_sent:\s+%{NUMBER:rows_sent:int}\s+Rows_examined:\s+%{NUMBER:rows_examined:int}\n\s*(?:use %{DATA:database};\s*\n)?SET\s+timestamp=%{NUMBER:timestamp};\n\s*(?<sql>(?<action>\w+)\b.*;)\s*(?:\n#\s+Time)?.*$" ]

            remove_field => ["message"] #删除原始日志,我试过写在mutate中,发现不起作用
}
            #无use database语句
          grok {
            match => [ "message" , "^#\s+User@Host:\s+%{USER:user}\[[^\]]+\]\s+@\s+(?:(?<clienthost>\S*) )?\[(?:%{IPV4:clientip})?\]\s+Id:\s+%{NUMBER:row_id:int}\n#\s+Query_time:\s+%{NUMBER:query_time:float}\s+Lock_time:\s+%{NUMBER:lock_time:float}\s+Rows_sent:\s+%{NUMBER:rows_sent:int}\s+Rows_examined:\s+%{NUMBER:rows_examined:int}\nSET\s+timestamp=%{NUMBER:timestamp};\n\s*(?<sql>(?<action>\w+)\b.*;)\s*(?:\n#\s+Time)?.*$" ]
           remove_field => ["message"]  #删除原始日志,我试过写在mutate中,发现不起作用
    }
        date {
            match => ["timestamp_mysql", "UNIX"]
            target => "@timestamp"
        }
       mutate {
            remove_field => "@version"  #删除filebeat传输过来的无用字段,可以视情况删除
       }
    }
```

> 也可以将2个match语句写入到一个grok中,但是我试过好像不行,有些慢日志并不能正常解析



实际上无论是grok debbuger在线工具还是Kibana的dev tool均提供了grok的在线调试工具,可以检测和调试grok的正则匹配.例如使用上述中的grok正则检测一下是否能正常匹配到前文提到的mysql原始日志数据.可以使用kibana的dev tool工具中的Grok Debugger工具来校验:

![image-20200729142559717](https://img2.jesse.top/image-20200729142559717.png)

上面的结构化数据输出中可以看到grok正则能正常解析原始日志中的数据,并且以json格式将日志内容映射给各字段.

---

###  Kibana展示

我尝试在kibana中使用图形化展示query_time(也就是SQL执行时间)最长的TOP5的SQL语句,制作成可视化图表,方便动态展示.但是发现可视化的聚合图形并不能满足这个需求.

但是Kibana的Discover界面通过选定字段,也能对query_time进行排序.例如下面的截图中先选定`query_time`和`sql`这2个字段,然后再对`query_time`进行排序(在右边的query_time字段下有个倒三角形表示倒序排序).

![image-20200729143159670](https://img2.jesse.top/image-20200729143159670.png)

然后将这个discover的筛选结果保存到Dashboard中,方便以后查看:

![image-20200729143309655](https://img2.jesse.top/image-20200729143309655.png)

