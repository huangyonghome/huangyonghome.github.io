---
title: Nginx limit_req_zone限流实战
date: 2018-08-3 11:59:58
tags:  [nginx, web ]
categories: [Linux-Web]
comments: true
copyright: true
---



## Nginx limit_req_zone限流实战

### 背景

昨天在公司对一个用户登陆的接口进行限流压测.利用Nginx自带的limit_req_zone模块.关于这个模块的介绍在之前的笔记中有写,所以就不再赘述.

---

### 系统拓扑

后端应用软件: PHP 版本5.6

应用服务器: 32核64G. 4台

数据库服务器: 56核225G . 1台

数据库软件: mysql 5.7 

流量入口: 阿里云SLB ..最大QPS和连接数:1W.(应用服务器在该SLB下)

---



### 限流策略

每台应用服务器每秒限制1200个QPS,一共4台服务器,每秒请求4800QPS

---

### Nginx限流配置

* nginx.conf配置文件:

```
http {
...
limit_req_zone $host zone=one:512m rate=1200r/s;
...
}
```

* hsq-open-api.conf虚拟主机配置文件

```
location ~ .*\.(php|php5)?$ {
...
limit_req zone=one burst=5 nodelay;
...
}
```

---

<!--more-->

### 压测记录

1:

压测配置5000个请求每秒,但是实际上只到3000个请求就挂了.

### 修改配置

```
将burst改成1200
```



2.

压测8000个请求每秒 ,大部分都能成功,看起来好像限流没有生效



### 修改配置

```
去掉nodelay关键字,实现精准的限流策略..nodelay表示不延时处理用户请求,.同时将burst改成100个.这样每秒的允许请求数量就是rate+burst=1300.再乘以4.将近5200个:

location ~ .*\.(php|php5)?$ {
...
limit_req zone=one burst=100;
...
}
```



3. 

这次压测效果好很多,压测6000/s的请求.成功率在5500左右.但是在压测8000/s的用户请求时,成功率仍然能到6500左右

## 修改配置

```
#rate改成1000,burst改成50.这样单台服务器的最大流量是1050/s. 总共是4200/s
http {
...
limit_req_zone $host zone=one:512m rate=1000r/s;
...
}

location ~ .*\.(php|php5)?$ {
...
limit_req zone=one burst=50;
...
}
```



4.

这次压测无论是6000/s请求场景还是8000/s的请求场景下,只有5000个左右的成功率.其余请求返回503错误.符合限流策略要求.



---



### 总结:

Nginx自带的限流模块不是非常精准,需要反复调试,得出个大概的限流范围.

