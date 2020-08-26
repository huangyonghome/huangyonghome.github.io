---
title:  Kibana图表制作
date: 2020-08-25 22:59:58
tags:  elk
categories: elk
comments: true
copyright: true
---



## Kibana图表制作

kibana的可视化可以制作各种统计分析图表,然后合并展示到dashbord中.下面介绍一些常用的nginx的访问图表.

> 在kibana7.7版本中可以配置kibana web界面的中文语言.

编辑kibana配置文件,`/etc/kibana/kibana.yml`,加入以下配置: 

`i18n.locale: "zh-CN"`

#### 一.统计过去XXX时间的访问量

在可视化界面,添加仪表盘图表.指定nginx的openapi access访问日志索引.

无需进行任何配置,选择右上角的时间范围,会显示计数,也就是日志量的计数

![image-20200723154706247](https://img2.jesse.top/image-20200723154706247.png)



<!--more-->

### openapi-nginx流量图

![image-20200806155157188](https://img2.jesse.top/image-20200806155157188.png)

#### 添加城市访问地图

在`map`界面新建一个地图.在`road map`的地图上添加图层.

![image-20200723155049422](https://img2.jesse.top/image-20200723155049422.png)

选择数据源.选择文档类型:

![image-20200723160002287](https://img2.jesse.top/image-20200723160002287.png)

选择索引后,选择`geoip.location`字段,此时客户端地图分布自动展现出现,而且会自动计数

![image-20200723160127185](https://img2.jesse.top/image-20200723160127185.png)



#### Nginx状态码统计图

添加饼图,使用`status`指标来统计各状态码的次数.

![image-20200723163110993](https://img2.jesse.top/image-20200723163110993.png)

> 如果没有status字段,需要去刷新索引

####  Nginx访问客户端TOP5

添加垂直条形图.使用geoip关键词统计各IP的访问次数,并且按降序排序

![image-20200723163406793](https://img2.jesse.top/image-20200723163406793.png)



####  nginx请求URL的TOP5

添加数据图表,使用request关键词统计各URL的访问次数,并按降序排序

![image-20200723163554578](https://img2.jesse.top/image-20200723163554578.png)



####  nginx的cost请求时间TOP10

添加垂直条形图.使用responsetime关键词做聚合,统计请求最慢的cost时间

![image-20200723163717829](https://img2.jesse.top/image-20200723163717829.png)

---

####  在Dashboard中将多个图表聚合成一个大盘

![image-20200723163906182](https://img2.jesse.top/image-20200723163906182.png)

​                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              